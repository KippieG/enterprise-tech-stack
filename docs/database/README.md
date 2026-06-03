# Database — MS SQL Server / T-SQL

## Overzicht

Microsoft SQL Server is de relationele database engine achter de meeste enterprise Microsoft-stack toepassingen. T-SQL (Transact-SQL) is de dialect waarmee je queries, stored procedures, views, triggers en meer schrijft. Goed T-SQL schrijven is een kernvaardigheid voor elke backend developer in deze stack.

---

## Fundamenten

### Data Types

```sql
-- Meest gebruikte types
INT, BIGINT, SMALLINT           -- Gehele getallen
DECIMAL(18,4), FLOAT            -- Decimalen (gebruik DECIMAL voor financiën!)
NVARCHAR(100), NVARCHAR(MAX)    -- Unicode tekst
VARCHAR(100)                    -- ASCII tekst (vermijd bij internationaal gebruik)
BIT                             -- Boolean (0 / 1)
DATETIME2(7)                    -- Datum + tijd (nauwkeuriger dan DATETIME)
DATE                            -- Alleen datum
UNIQUEIDENTIFIER                -- GUID
```

---

## Query Schrijven

### Basispatroon — SELECT met joins

```sql
SELECT
    o.Id,
    o.OrderNumber,
    o.OrderDate,
    c.Name AS CustomerName,
    SUM(ol.Quantity * ol.UnitPrice) AS TotalAmount
FROM Orders o
INNER JOIN Customers c ON c.Id = o.CustomerId
INNER JOIN OrderLines ol ON ol.OrderId = o.Id
WHERE o.Status = 'Open'
  AND o.OrderDate >= DATEADD(DAY, -30, GETDATE())
GROUP BY
    o.Id, o.OrderNumber, o.OrderDate, c.Name
HAVING SUM(ol.Quantity * ol.UnitPrice) > 1000
ORDER BY o.OrderDate DESC;
```

### CTE — Common Table Expressions

```sql
WITH RecentOrders AS (
    SELECT
        CustomerId,
        COUNT(*) AS OrderCount,
        SUM(TotalAmount) AS TotalSpent
    FROM Orders
    WHERE OrderDate >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY CustomerId
),
RankedCustomers AS (
    SELECT
        c.Id,
        c.Name,
        ro.OrderCount,
        ro.TotalSpent,
        RANK() OVER (ORDER BY ro.TotalSpent DESC) AS SpendingRank
    FROM Customers c
    INNER JOIN RecentOrders ro ON ro.CustomerId = c.Id
)
SELECT *
FROM RankedCustomers
WHERE SpendingRank <= 10;
```

### Window Functions

```sql
SELECT
    OrderId,
    ProductCode,
    Quantity,
    -- Lopend totaal per order
    SUM(Quantity) OVER (PARTITION BY OrderId ORDER BY Id) AS RunningQty,
    -- Rang binnen order op hoeveelheid
    RANK() OVER (PARTITION BY OrderId ORDER BY Quantity DESC) AS QtyRank,
    -- Vorige regel
    LAG(Quantity, 1, 0) OVER (PARTITION BY OrderId ORDER BY Id) AS PrevQty
FROM OrderLines;
```

---

## Stored Procedures

```sql
CREATE OR ALTER PROCEDURE usp_CreateOrder
    @OrderNumber    NVARCHAR(50),
    @CustomerId     INT,
    @DeliveryDate   DATE,
    @CreatedById    INT,
    @NewOrderId     INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    -- Validatie
    IF NOT EXISTS (SELECT 1 FROM Customers WHERE Id = @CustomerId AND IsActive = 1)
    BEGIN
        RAISERROR('Klant bestaat niet of is inactief.', 16, 1);
        RETURN;
    END

    IF EXISTS (SELECT 1 FROM Orders WHERE OrderNumber = @OrderNumber)
    BEGIN
        RAISERROR('Ordernummer bestaat al.', 16, 1);
        RETURN;
    END

    -- Insert
    INSERT INTO Orders (OrderNumber, CustomerId, DeliveryDate, Status, CreatedById, CreatedAt)
    VALUES (@OrderNumber, @CustomerId, @DeliveryDate, 'Draft', @CreatedById, SYSDATETIME());

    SET @NewOrderId = SCOPE_IDENTITY();
END;
GO

-- Aanroepen vanuit C#:
-- EXEC usp_CreateOrder @OrderNumber='ORD-2026-001', @CustomerId=5, ...
```

---

## Indexen

### Clustered vs Non-Clustered

```sql
-- Clustered index: de fysieke volgorde van de tabel (1 per tabel, meestal PK)
CREATE CLUSTERED INDEX CIX_Orders_Id ON Orders(Id);

-- Non-clustered: aparte index structuur
CREATE NONCLUSTERED INDEX NIX_Orders_Status_Date
    ON Orders(Status, OrderDate DESC)
    INCLUDE (CustomerId, TotalAmount);  -- Covering index — vermijdt key lookup
```

### Index best practices
- Indexeer kolommen die veel in `WHERE`, `JOIN ON` en `ORDER BY` voorkomen
- Gebruik `INCLUDE` om key lookups te vermijden (covering index)
- Vermijd te veel indexen op schrijf-intensieve tabellen
- Monitor met `sys.dm_db_missing_index_details` en `sys.dm_db_index_usage_stats`

---

## Transacties

```sql
BEGIN TRY
    BEGIN TRANSACTION;

    UPDATE Inventory SET Quantity = Quantity - @Qty
    WHERE ProductId = @ProductId;

    IF (SELECT Quantity FROM Inventory WHERE ProductId = @ProductId) < 0
    BEGIN
        RAISERROR('Onvoldoende voorraad.', 16, 1);
    END

    INSERT INTO Shipments (OrderId, ProductId, Quantity, ShippedAt)
    VALUES (@OrderId, @ProductId, @Qty, SYSDATETIME());

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION;

    INSERT INTO ErrorLog (Message, ErrorLine, StoredProcedure, OccurredAt)
    VALUES (ERROR_MESSAGE(), ERROR_LINE(), ERROR_PROCEDURE(), SYSDATETIME());

    THROW;
END CATCH;
```

---

## Query Optimalisatie

### Uitvoeringsplan lezen

```sql
-- Schakel in voor query analyse
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Of gebruik:
-- CTRL+M in SSMS voor grafisch uitvoeringsplan
-- Zoek naar: Table Scan (slecht), Index Seek (goed), Key Lookup (verbeterbaar)
```

### Veelgemaakte performantieproblemen

| Probleem | Oplossing |
|---------|-----------|
| `SELECT *` | Selecteer alleen benodigde kolommen |
| `LIKE '%zoekterm'` | Gebruik Full-Text Search of heroverweeg het datamodel |
| Impliciet type conversie | Zorg voor matching datatypes in joins/where |
| N+1 queries | Gebruik joins of `CROSS APPLY` |
| Missing index | Voeg covering index toe |
| Geen `NOLOCK` hint misbruik | Gebruik snapshot isolation i.p.v. dirty reads |

---

## Schema Design Tips

```sql
-- Altijd:
-- 1. Primary key op elke tabel
-- 2. Audit kolommen
-- 3. Soft delete (IsDeleted) in plaats van hard delete
-- 4. Foreign key constraints voor referentiële integriteit

CREATE TABLE Orders (
    Id              INT IDENTITY(1,1)   NOT NULL,
    OrderNumber     NVARCHAR(50)        NOT NULL,
    CustomerId      INT                 NOT NULL,
    Status          NVARCHAR(20)        NOT NULL DEFAULT 'Draft',
    IsDeleted       BIT                 NOT NULL DEFAULT 0,
    CreatedAt       DATETIME2(0)        NOT NULL DEFAULT SYSDATETIME(),
    CreatedById     INT                 NOT NULL,
    ModifiedAt      DATETIME2(0)            NULL,
    ModifiedById    INT                     NULL,

    CONSTRAINT PK_Orders PRIMARY KEY (Id),
    CONSTRAINT UQ_Orders_OrderNumber UNIQUE (OrderNumber),
    CONSTRAINT FK_Orders_CustomerId FOREIGN KEY (CustomerId) REFERENCES Customers(Id),
    CONSTRAINT CHK_Orders_Status CHECK (Status IN ('Draft','Open','Confirmed','Shipped','Closed','Cancelled'))
);
```

---

## Nuttige system views

```sql
-- Welke queries verbruiken de meeste CPU?
SELECT TOP 10
    total_worker_time / execution_count AS avg_cpu_time,
    execution_count,
    SUBSTRING(text, statement_start_offset/2, 200) AS query_text
FROM sys.dm_exec_query_stats
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
ORDER BY avg_cpu_time DESC;

-- Welke tabellen hebben de meeste reads?
SELECT
    OBJECT_NAME(i.object_id) AS TableName,
    SUM(user_seeks + user_scans + user_lookups) AS TotalReads
FROM sys.dm_db_index_usage_stats i
WHERE database_id = DB_ID()
GROUP BY i.object_id
ORDER BY TotalReads DESC;
```

---

*[← Terug naar overzicht](../../README.md)*
