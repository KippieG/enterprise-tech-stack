# WMS — WACS (Warehouse Management System)

## Overzicht

WACS is een Warehouse Management System dat alle warehouseprocessen beheert: van ontvangst en opslag tot picking, verpakking en verzending. Het is het hart van de logistieke operatie en integreert met TMS (TAS) voor transport en ERP (Business Central) voor voorraad en facturatie.

---

## Kernprocessen

```
INBOUND (Ontvangst)
    Leverancier → Aanmelding (ASN) → Ontvangst op dock
    → Kwaliteitscontrole → Putaway naar locatie
    → Bevestiging aan ERP (goederenontvangst)

OPSLAG (Storage)
    Locatiebeheer → Zone indeling → Slimme opslag
    (snellopers dicht bij expeditie, zwaar onderaan)

OUTBOUND (Verzending)
    Verkooporder → Pickorder aanmaken
    → Picking → Verpakking → Labeling
    → Laadlijst → Verlading → Transport aan TAS

VOORRAADBEHEER
    Cyclustellingen → Correcties → FIFO/FEFO
    → Voorraadrapportage naar ERP
```

---

## Datamodel (conceptueel)

```
Warehouse
├── Zones (A-koeling, B-normaal, C-gevaarlijk, ...)
│   └── Aisles → Sections → Levels → Locations
│       └── Location: Code, Type, Capacity (vol/kg), Status

Articles (Artikelen)
├── ArticleCode (= ProductCode in ERP/TMS)
├── Description
├── UOM (eenheden: ST, KG, PAL, ...)
├── StorageConditions (temp, vochtigheid)
├── DangerousGoods (ADR klasse)
└── GTIN / Barcode

Stock (Voorraad per locatie)
├── Location
├── Article
├── Lot / Batch
├── ExpiryDate
├── Quantity
└── Status (Vrij / Geblokkeerd / In picking)

ReceiptOrder (Ontvangst)
├── ReceiptNumber
├── SupplierCode
├── ExpectedDate / ActualDate
└── Lines → Article, ExpectedQty, ReceivedQty, Lot, ExpiryDate

PickOrder (Uitgifte)
├── PickOrderNumber
├── Reference (verkooporder/transportorder)
├── Priority
├── Status (Nieuw / In picking / Gereed / Verzonden)
└── Lines
    ├── Article, Quantity
    ├── Assigned Location
    └── Picked Quantity + Picker
```

---

## Locatiebeheer

### Locatiestructuur

```
WH-A-01-01-01
│   │  │  │  └── Vak (01 = onderste)
│   │  │  └───── Sectie (01 = eerste sectie in gang)
│   │  └──────── Gang (01 = eerste gang)
│   └─────────── Zone (A = normale opslag)
└─────────────── Warehouse (WH = hoofdmagazijn)
```

### Slimme putaway logica

```csharp
public class PutawayEngine
{
    public async Task<Location> SuggestLocationAsync(Article article, decimal quantity)
    {
        // 1. Zoek bestaande stock van zelfde artikel (consolidatie)
        var existingLocation = await _stockRepo
            .GetLocationsWithArticle(article.Code)
            .Where(l => l.HasCapacityFor(quantity))
            .FirstOrDefaultAsync();

        if (existingLocation != null)
            return existingLocation;

        // 2. Kies zone op basis van artikel eigenschappen
        var zone = article.RequiresCooling ? "COOL"
                 : article.IsDangerousGoods ? "ADR"
                 : article.IsHighTurnover ? "A"    // Snelloperszone, dicht bij expeditie
                 : "B";

        // 3. Selecteer lege locatie (LIFO voor leegmaken gangpaden)
        return await _locationRepo
            .GetEmptyLocationsInZone(zone)
            .Where(l => l.WeightCapacity >= article.WeightPerUnit * quantity)
            .OrderBy(l => l.DistanceFromDock)
            .FirstOrDefaultAsync()
            ?? throw new NoLocationAvailableException(zone);
    }
}
```

---

## Picking

### Pickwave strategie

```
Wave picking (batch meerdere orders tegelijk):
    Voordeel: minder kilometers in magazijn
    Nadeel: orders moeten samen gesorteerd worden

Order picking (één order per keer):
    Voordeel: eenvoudig, geen sortering nodig
    Nadeel: meer kilometers bij hoge ordervolumes

Zone picking (picker blijft in zijn zone):
    Voordeel: specialisatie, minder fouten
    Nadeel: orders moeten samengevoegd worden (merge point)
```

### Pick volgorde optimalisatie

```sql
-- Genereer picklijst geoptimaliseerd op looproute
SELECT
    pol.PickOrderLineId,
    pol.ArticleCode,
    pol.Description,
    pol.QuantityToPick,
    s.LocationCode,
    l.Aisle,
    l.Section,
    l.Level,
    s.LotNumber,
    s.ExpiryDate,
    -- Sortering: gang, dan sectie zigzag, dan level
    ROW_NUMBER() OVER (
        ORDER BY
            l.Aisle,
            CASE WHEN CAST(l.Aisle AS INT) % 2 = 0
                 THEN l.Section           -- even gang: van voor naar achter
                 ELSE 999 - l.Section     -- oneven gang: van achter naar voor
            END,
            l.Level
    ) AS PickSequence
FROM PickOrderLines pol
INNER JOIN Stock s ON s.ArticleCode = pol.ArticleCode
    AND s.Status = 'Free'
    AND (s.ExpiryDate IS NULL OR s.ExpiryDate > GETDATE())  -- FEFO
INNER JOIN Locations l ON l.LocationCode = s.LocationCode
WHERE pol.PickOrderId = @PickOrderId
ORDER BY PickSequence;
```

---

## Integratie WACS ↔ TAS (Transport)

### Inbound: TAS stuurt pickorder

```http
POST /api/wacs/pickorders
Authorization: Bearer {token}
Content-Type: application/json

{
  "reference": "TAS-ORD-2026-001",
  "requestedReadyBy": "2026-06-04T06:00:00",
  "priority": "High",
  "deliveryAddress": {
    "name": "Klant BV",
    "street": "Havenstraat 1",
    "city": "Antwerpen",
    "country": "BE"
  },
  "lines": [
    {
      "lineNumber": 1,
      "articleCode": "ART-001",
      "quantity": 10,
      "uom": "ST"
    }
  ]
}
```

### Outbound: WACS bevestigt gereed

```csharp
public async Task NotifyTASOrderReadyAsync(PickOrder pickOrder)
{
    var notification = new TASReadyNotification
    {
        TASReference = pickOrder.TASReference,
        WACSReference = pickOrder.PickOrderNumber,
        ReadyAt = DateTime.UtcNow,
        ActualLines = pickOrder.Lines.Select(l => new TASReadyLine
        {
            ArticleCode = l.ArticleCode,
            QuantityPicked = l.QuantityPicked,
            LotNumber = l.AssignedLot,
            LocationPicked = l.PickedFromLocation
        }).ToList(),
        TotalWeight = pickOrder.Lines.Sum(l => l.WeightPicked),
        TotalVolume = pickOrder.Lines.Sum(l => l.VolumePicked)
    };

    await _tasClient.PostReadyNotificationAsync(notification);
}
```

---

## Integratie WACS ↔ Business Central

### Voorraadmutaties synchroniseren

```csharp
public class StockSyncService
{
    // Elke mutatie in WACS wordt bijgehouden en gesynchroniseerd naar BC
    public async Task SyncMutationToBCAsync(StockMutation mutation)
    {
        var bcEntry = new BCItemJournalLine
        {
            ItemNo = mutation.ArticleCode,
            LocationCode = "WAREHOUSE",
            EntryType = mutation.Type switch
            {
                MutationType.Receipt => BCEntryType.PositiveAdjustment,
                MutationType.Shipment => BCEntryType.NegativeAdjustment,
                MutationType.Correction => mutation.Quantity > 0
                    ? BCEntryType.PositiveAdjustment
                    : BCEntryType.NegativeAdjustment,
                _ => throw new ArgumentException("Onbekend mutatietype")
            },
            Quantity = Math.Abs(mutation.Quantity),
            LotNo = mutation.LotNumber,
            ExpirationDate = mutation.ExpiryDate,
            ExternalDocumentNo = mutation.Reference
        };

        await _bcClient.PostItemJournalLineAsync(bcEntry);
    }
}
```

---

## Voorraadbeheer & Cyclustellingen

### FIFO / FEFO principe

- **FIFO** (First In First Out): oudste voorraad eerst weg — gebruikt voor niet-bederfelijke goederen
- **FEFO** (First Expired First Out): kortste houdbaarheidsdatum eerst — verplicht voor voedsel, farma

```sql
-- FEFO: welke lot bij het picken gebruiken?
SELECT TOP 1
    s.LotNumber,
    s.ExpiryDate,
    s.LocationCode,
    s.Quantity
FROM Stock s
WHERE s.ArticleCode = @ArticleCode
  AND s.Status = 'Free'
  AND s.Quantity >= @QuantityRequired
  AND (s.ExpiryDate IS NULL OR s.ExpiryDate >= DATEADD(DAY, @MinRemainingDays, GETDATE()))
ORDER BY
    COALESCE(s.ExpiryDate, '9999-12-31') ASC,  -- Kortste datum eerst
    s.ReceiptDate ASC;                           -- Bij gelijke datum: oudste ontvangst
```

### Cyclustellingplan

```sql
-- Genereer telplan: artikelen met hoogste omzet of laag betrouwbaarheidsscore eerst
SELECT TOP 50
    a.ArticleCode,
    a.Description,
    s.LocationCode,
    s.Quantity AS SystemQuantity,
    s.LastCountDate,
    DATEDIFF(DAY, s.LastCountDate, GETDATE()) AS DaysSinceCount,
    mv.MovementsLast30Days,
    -- Score: hoe hoger, hoe urgenter om te tellen
    (DATEDIFF(DAY, s.LastCountDate, GETDATE()) * 0.3
     + mv.MovementsLast30Days * 0.5
     + CASE WHEN s.StockAccuracy < 0.98 THEN 50 ELSE 0 END) AS CountPriority
FROM Articles a
INNER JOIN Stock s ON s.ArticleCode = a.ArticleCode
LEFT JOIN (
    SELECT ArticleCode, COUNT(*) AS MovementsLast30Days
    FROM StockMutations
    WHERE MutationDate >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ArticleCode
) mv ON mv.ArticleCode = a.ArticleCode
WHERE a.IsActive = 1
ORDER BY CountPriority DESC;
```

---

## KPI's voor Warehouse

| KPI | Definitie | Streefwaarde |
|-----|-----------|--------------|
| Order Accuracy | % orders zonder fouten gepicked | > 99.5% |
| On-Time Dispatch | % orders klaar voor transport op tijd | > 98% |
| Stock Accuracy | % locaties correct bij telling | > 99% |
| Picks per uur | Productiviteit per picker | Benchmark-afhankelijk |
| FEFO compliance | % pickorders die FEFO respecteren | 100% |
| Dock-to-Stock tijd | Tijd van ontvangst tot opgeslagen | < 4u |

---

## Best Practices

- **Barcode/RF-scanning** op elke stap: ontvangst, putaway, picking, verzending — geen handmatige invoer
- **Locatie confirmatie**: picker scant locatie én artikel — dubbele verificatie
- **Exception management**: geblokkeerde voorraad, kwaliteitshold en quarantaine altijd zichtbaar
- **Real-time dashboards**: zie actuele warehousestatus op elk moment (bezetting, openstaande picks, dock status)
- **Integratielog**: elke bericht van/naar TAS en BC bijhouden voor troubleshooting

---

*[← Terug naar overzicht](../../README.md)*
