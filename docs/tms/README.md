# TMS — TAS (Transport Management System)

## Overzicht

TAS is een Transport Management System dat transportorders, planning, chauffeurs, voertuigen en klantcommunicatie beheert. Het vormt de operationele kern voor transportbedrijven en logistieke dienstverleners.

---

## Kernfunctionaliteiten

| Module | Doel |
|--------|------|
| **Orderbeheer** | Transportopdrachten registreren en beheren |
| **Planning** | Ritten en routes optimaliseren, chauffeurs en voertuigen toewijzen |
| **Tracking** | Real-time locatie en statusupdates van voertuigen |
| **Klantportal** | Klanten volgen hun zendingen |
| **Facturatie** | Transportkosten berekenen en facturen genereren |
| **Rapportages** | KPI's, tijdigheid, kosten per route |

---

## Datamodel (conceptueel)

```
TransportOrder
├── OrderHeader
│   ├── OrderNumber (uniek)
│   ├── CustomerCode → ref. Customers
│   ├── Status (Nieuw / Gepland / In transit / Afgeleverd / Gefactureerd)
│   ├── RequestedPickupDate
│   └── RequestedDeliveryDate
│
├── OrderLines (per artikel/collo)
│   ├── ProductCode
│   ├── Description
│   ├── Quantity
│   ├── Weight (kg)
│   ├── Volume (m³)
│   └── DangerousGoods (ADR klasse)
│
├── Pickup Address
│   └── Name, Street, PostalCode, City, Country, ContactPerson, TimeWindow
│
└── Delivery Address
    └── Name, Street, PostalCode, City, Country, ContactPerson, TimeWindow

Trip (Rit)
├── TripNumber
├── Driver → ref. Drivers
├── Vehicle → ref. Vehicles
├── DepartureTime / ArrivalTime
└── Stops (geordend)
    ├── StopType (Pickup / Delivery)
    ├── OrderLine refs
    ├── PlannedTime
    └── ActualTime + StatusCode
```

---

## Integratie met externe systemen

### TAS → WMS (WACS)

Wanneer een transportorder klaar staat voor picking:

```csharp
public class TASToWACSIntegration
{
    public async Task SendPickupOrderToWMS(TransportOrder order)
    {
        var wacsPayload = new WACSPickupOrder
        {
            Reference = order.OrderNumber,
            PickupDate = order.RequestedPickupDate,
            CustomerCode = order.CustomerCode,
            Lines = order.Lines.Select(l => new WACSPickupLine
            {
                ProductCode = l.ProductCode,
                Quantity = l.Quantity,
                Weight = l.Weight
            }).ToList()
        };

        await _wacsClient.CreatePickupOrderAsync(wacsPayload);
    }
}
```

### TAS → Business Central (Facturatie)

Na aflevering stuurt TAS de gegevens naar BC voor facturatie:

```csharp
public async Task PostCompletedTripToBC(Trip trip)
{
    foreach (var order in trip.CompletedOrders)
    {
        var salesOrder = new BCSalesOrderCommand
        {
            ExternalDocumentNo = order.OrderNumber,
            SellToCustomerNo = order.CustomerCode,
            RequestedDeliveryDate = order.ActualDeliveryDate,
            Lines = CalculateTransportCosts(order, trip)
        };

        await _bcClient.CreateSalesOrderAsync(salesOrder);
    }
}

private List<BCSalesLine> CalculateTransportCosts(TransportOrder order, Trip trip)
{
    var lines = new List<BCSalesLine>();

    // Basistarief
    lines.Add(new BCSalesLine
    {
        ItemNo = "TRANSPORT-BASE",
        Quantity = 1,
        UnitPrice = _tariffEngine.GetBaseTariff(order.CustomerCode, trip.Vehicle.Type)
    });

    // Gewichtstoeslag
    if (order.TotalWeight > 500)
        lines.Add(new BCSalesLine
        {
            ItemNo = "TRANSPORT-WEIGHT",
            Quantity = order.TotalWeight,
            UnitPrice = _tariffEngine.GetWeightRate(order.CustomerCode)
        });

    return lines;
}
```

---

## Planning & Route Optimalisatie

### Planningsprincipes

```
Input:
- Openstaande transportorders (pickup + delivery adressen, tijdvensters)
- Beschikbare chauffeurs (rijlicenties, rijtijden, locatie)
- Beschikbare voertuigen (capaciteit, gewicht, koelvereisten)

Constraints:
- Chauffeur rijtijden (EU regelgeving: max 9u rijden, 11u rusttijd)
- Tijdvensters klant (levering tussen 08:00–12:00)
- Voertuigcapaciteit (volume, gewicht, ADR)
- Geografische beperkingen (milieuzones, hoogtebeperkingen)

Output:
- Geoptimaliseerde ritten per chauffeur/voertuig
- Volgorde van stops
- Geschatte aankomsttijden (ETA)
```

### ETD/ETA berekening

```sql
-- Bereken verwachte aankomsttijden per stop
WITH TripStops AS (
    SELECT
        t.TripId,
        t.TripNumber,
        s.StopSequence,
        s.StopType,
        s.Address_City,
        s.PlannedTime,
        s.ActualArrivalTime,
        s.ActualDepartureTime,
        LAG(s.ActualDepartureTime) OVER (
            PARTITION BY t.TripId
            ORDER BY s.StopSequence
        ) AS PreviousDeparture,
        DATEDIFF(MINUTE,
            LAG(s.ActualDepartureTime) OVER (PARTITION BY t.TripId ORDER BY s.StopSequence),
            s.ActualArrivalTime
        ) AS DriveTimeMinutes
    FROM Trips t
    INNER JOIN TripStops s ON s.TripId = t.TripId
    WHERE t.TripDate = CAST(GETDATE() AS DATE)
)
SELECT
    TripNumber,
    StopSequence,
    StopType,
    Address_City,
    PlannedTime,
    ActualArrivalTime,
    DriveTimeMinutes,
    CASE
        WHEN ActualArrivalTime <= PlannedTime THEN 'Op tijd'
        WHEN ActualArrivalTime IS NULL THEN 'In transit'
        ELSE CONCAT('Te laat: +', DATEDIFF(MINUTE, PlannedTime, ActualArrivalTime), ' min')
    END AS Status
FROM TripStops
ORDER BY TripId, StopSequence;
```

---

## Track & Trace

### Status lifecycle

```
Nieuw → Gepland → Vertrokken → Bij pickup → Geladen → In transit
→ Bij aflevering → Afgeleverd ✓
              → Niet thuis → Herlevering gepland
              → Geweigerd → Retour in verwerking
```

### Klant notificaties

```csharp
public class TrackTraceNotificationService
{
    public async Task NotifyCustomerAsync(TransportOrder order, TrackTraceStatus status)
    {
        var message = status switch
        {
            TrackTraceStatus.OutForDelivery =>
                $"Je zending {order.OrderNumber} is onderweg. " +
                $"Verwachte levering: {order.ETA:HH:mm}.",
            TrackTraceStatus.Delivered =>
                $"Je zending {order.OrderNumber} is afgeleverd op {DateTime.Now:dd/MM HH:mm}.",
            TrackTraceStatus.DeliveryAttemptFailed =>
                $"Levering van {order.OrderNumber} mislukt. We nemen contact op voor herplannen.",
            _ => null
        };

        if (message is not null)
        {
            await _emailService.SendAsync(order.CustomerEmail, "Status update zending", message);
            await _smsService.SendAsync(order.CustomerPhone, message);
        }
    }
}
```

---

## KPI's voor Transport

| KPI | Definitie | Streefwaarde |
|-----|-----------|--------------|
| On-Time Delivery (OTD) | % leveringen op afgesproken tijdstip | > 95% |
| First Attempt Delivery | % succesvol bij eerste poging | > 90% |
| Km per zending | Gemiddeld aantal km per afgeleverde order | Minimaliseer |
| Vrachtbenutting | Gemiddeld % van capaciteit benut | > 80% |
| CO₂ per ton·km | Uitstoot per tonkm | Verbetering YoY |

---

## Best Practices voor TMS Integraties

- Gebruik **event-driven architectuur**: TAS publiceert statuswijzigingen op een bus, WMS en BC consumeren die — geen polling
- **Idempotente verwerking**: zelfde bericht twee keer ontvangen mag geen dubbele records creëren
- **Correlation IDs**: traceer een order door TAS, WMS en BC met één unieke ID
- **Master data sync**: klanten, adressen en producten hebben één master (BC) — andere systemen synchroniseren
- Documenteer **alle integratiepunten** in een interface-catalogus

---

*[← Terug naar overzicht](../../README.md)*
