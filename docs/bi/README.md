# BI — Power BI · T-SQL · Excel

## Overzicht

Business Intelligence verbindt ruwe data met zakelijke beslissingen. Power BI is de rapportage- en visualisatietool van Microsoft, T-SQL levert de data aan, en Excel is de universele tool voor ad-hoc analyse.

---

## Power BI

### Architectuur

```
Databron (SQL Server)
    ↓
Power Query (M) — data transformeren & opschonen
    ↓
Data Model (tabellen + relaties)
    ↓
DAX (berekeningen & measures)
    ↓
Visuals (grafieken, tabellen, kaarten)
    ↓
Rapport → Dashboard → Power BI Service → Teams/SharePoint
```

### Data Model best practices

- **Star schema**: feitentabellen in het midden, dimensietabellen rondom
- Gebruik **surrogate keys** (INT) als join-sleutels, niet business keys
- Zet **datum-dimensie** apart — nooit `OrderDate` direct op as

```
DimDate ──────┐
DimCustomer ──┤── FactOrders ──┬── DimProduct
DimEmployee ──┘                └── DimWarehouse
```

---

## DAX — Data Analysis Expressions

### Basismaatregelen

```dax
-- Totale omzet
Total Revenue = SUMX(FactOrders, FactOrders[Quantity] * FactOrders[UnitPrice])

-- Aantal unieke klanten
Unique Customers = DISTINCTCOUNT(FactOrders[CustomerId])

-- Gemiddelde orderwaarde
Avg Order Value = DIVIDE([Total Revenue], [Order Count])
```

### Tijdsintelligentie

```dax
-- Omzet vorig jaar (zelfde periode)
Revenue PY = CALCULATE([Total Revenue], SAMEPERIODLASTYEAR(DimDate[Date]))

-- Groei t.o.v. vorig jaar %
YoY Growth % = DIVIDE([Total Revenue] - [Revenue PY], [Revenue PY])

-- Year-to-date omzet
Revenue YTD = TOTALYTD([Total Revenue], DimDate[Date])

-- Lopende som
Revenue Running Total =
CALCULATE(
    [Total Revenue],
    FILTER(
        ALL(DimDate),
        DimDate[Date] <= MAX(DimDate[Date])
    )
)
```

### CALCULATE — het hart van DAX

```dax
-- Omzet alleen voor klanten in België
Belgium Revenue =
CALCULATE(
    [Total Revenue],
    DimCustomer[Country] = "Belgium"
)

-- Top 10 klanten (context filter)
Top 10 Customers Revenue =
CALCULATE(
    [Total Revenue],
    TOPN(10, DimCustomer, [Total Revenue])
)

-- Omzet exclusief huidige filter (voor % van totaal)
Revenue % of Total =
DIVIDE(
    [Total Revenue],
    CALCULATE([Total Revenue], ALL(DimCustomer))
)
```

---

## Power Query (M) — Data Transformatie

```m
// Voorbeeld: orders ophalen, opschonen en transformeren
let
    Source = Sql.Database("myserver", "MyDB"),
    Orders = Source{[Schema="dbo", Item="Orders"]}[Data],

    // Filter op actieve orders
    ActiveOrders = Table.SelectRows(Orders, each [IsDeleted] = false),

    // Kolommen selecteren
    Selected = Table.SelectColumns(ActiveOrders,
        {"Id", "OrderNumber", "OrderDate", "CustomerId", "TotalAmount", "Status"}),

    // Type instellen
    Typed = Table.TransformColumnTypes(Selected, {
        {"Id", Int64.Type},
        {"OrderDate", type date},
        {"TotalAmount", Currency.Type}
    }),

    // Berekende kolom toevoegen
    WithMonth = Table.AddColumn(Typed, "OrderMonth",
        each Date.ToText([OrderDate], "yyyy-MM"), type text)
in
    WithMonth
```

---

## T-SQL voor BI & Rapportage

### Rapportage views

```sql
-- Maak een view die de BI-tool rechtstreeks kan aanroepen
CREATE OR ALTER VIEW vw_SalesReport AS
SELECT
    o.Id AS OrderId,
    o.OrderNumber,
    CAST(o.OrderDate AS DATE) AS OrderDate,
    YEAR(o.OrderDate) AS OrderYear,
    MONTH(o.OrderDate) AS OrderMonth,
    DATENAME(MONTH, o.OrderDate) AS MonthName,
    DATEPART(QUARTER, o.OrderDate) AS OrderQuarter,
    c.Name AS CustomerName,
    c.Country,
    c.Segment AS CustomerSegment,
    p.ProductCode,
    p.ProductName,
    p.Category,
    ol.Quantity,
    ol.UnitPrice,
    ol.Quantity * ol.UnitPrice AS LineAmount,
    o.Status
FROM Orders o
INNER JOIN Customers c ON c.Id = o.CustomerId
INNER JOIN OrderLines ol ON ol.OrderId = o.Id
INNER JOIN Products p ON p.Id = ol.ProductId
WHERE o.IsDeleted = 0
  AND c.IsDeleted = 0;
GO
```

### Pivot voor rapportage

```sql
-- Omzet per maand per regio — gepivot
SELECT *
FROM (
    SELECT
        YEAR(OrderDate) AS Year,
        DATENAME(MONTH, OrderDate) AS Month,
        c.Region,
        ol.Quantity * ol.UnitPrice AS Amount
    FROM Orders o
    INNER JOIN OrderLines ol ON ol.OrderId = o.Id
    INNER JOIN Customers c ON c.Id = o.CustomerId
) AS SourceData
PIVOT (
    SUM(Amount)
    FOR Region IN ([Noord], [Oost], [Zuid], [West])
) AS PivotTable
ORDER BY Year, Month;
```

---

## Excel voor BI

### Power Query in Excel

- **Data → Get Data → From Database → SQL Server**
- Verbind met dezelfde views die Power BI ook gebruikt — één bron van waarheid
- Gebruik **Power Pivot** (gratis in Excel) voor eenvoudige data modellen

### Handige Excel formules voor BI

```excel
# XLOOKUP (moderne vervanger van VLOOKUP)
=XLOOKUP(A2, KlantenTabel[KlantId], KlantenTabel[KlantNaam], "Niet gevonden")

# Dynamische array — unieke waarden
=UNIQUE(B2:B100)

# Voorwaardelijk aggregeren
=SUMIFS(Bedrag, Regio, "Noord", Jaar, 2026)
=COUNTIFS(Status, "Open", Klant, A2)

# Dynamisch dashboard via FILTER
=FILTER(OrderenTabel, (OrderenTabel[Regio]=DropdownRegio) * (OrderenTabel[Jaar]=DropdownJaar))
```

---

## Best Practices

- **Één bron van waarheid**: alle rapporten koppelen aan dezelfde views of datasets
- **Incrementeel vernieuwen** in Power BI Service voor grote datasets
- Gebruik **Row-Level Security (RLS)** in Power BI zodat gebruikers alleen hun eigen data zien
- **Geen berekeningen in Power Query** die ook in DAX kunnen — DAX is sneller en dynamischer
- Gebruik **geïmporteerde data** voor historische rapporten, **DirectQuery** alleen als echt nodig

---

*[← Terug naar overzicht](../../README.md)*
