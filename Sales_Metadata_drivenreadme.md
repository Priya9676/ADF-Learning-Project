Initialized by Azure Data Factory!# Sales Data Project -- Simple Notes

## 1. What I Created

I created a metadata-driven Sales data pipeline using Azure Data
Factory, ADLS, Azure SQL, staging tables and data warehouse tables.

The main purpose is to process SalesOrder data in a reusable way and
support incremental loading.

------------------------------------------------------------------------

# 2. Main Entities

I have 4 entities:

``` text
Product
Customer
Store
SalesOrder
```

SalesOrder is the main Fact entity.

Product, Customer and Store are Dimension entities.

------------------------------------------------------------------------

# 3. Tables

## Metadata Tables

``` text
meta.SourceSystem
meta.EntityConfig
meta.ColumnMapping
meta.ValidationRule
meta.FileLog
meta.PipelineRunLog
meta.Watermark
```

### Simple purpose

-   `EntityConfig` → stores entity configuration.
-   `ColumnMapping` → stores column mapping.
-   `ValidationRule` → stores validation rules.
-   `Watermark` → stores the last processed date.
-   `FileLog` → stores file processing information.
-   `PipelineRunLog` → stores pipeline run information.

## Staging Tables

``` text
stg.SalesOrder
stg.Customer
stg.Product
stg.Store
stg.ErrorRows
stg.DuplicateLog
```

The source data is first copied into staging.

## Warehouse Tables

### Dimensions

``` text
dw.DimDate
dw.DimProduct
dw.DimCustomer
dw.DimStore
```

### Fact

``` text
dw.FactSales
```

`FactSales` stores the SalesOrder transactions.

------------------------------------------------------------------------

# 4. Master Pipeline

## `pl_MetaDriven_Master`

The Master pipeline is used for orchestration.

``` text
GetActiveEntities
       ↓
ForEachEntity
       ↓
Execute Child Pipeline
```

### What it does

1.  Gets active entities from `meta.EntityConfig`.
2.  Loops through the entities.
3.  Passes the `EntityId` to the Child pipeline.

### Simple meaning

> **Master Pipeline decides WHAT to process.**

Example:

``` text
Product
Customer
Store
SalesOrder
```

------------------------------------------------------------------------

# 5. Child Pipeline

## `pl_MetaDriven_ProcessEntity`

The Child pipeline performs the actual processing.

Current flow:

``` text
GetEntityConfig
       ↓
GetFiles
       ↓
Get_Watermark
       ↓
ForEachFile
       ↓
Copy_Current_File_To_Stage
       ↓
ProcessSalesOrder
       ↓
Update_Watermark
```

### Simple meaning

> **Child Pipeline decides HOW to process the selected entity.**

------------------------------------------------------------------------

# 6. GetEntityConfig

Reads the configuration for the selected entity from:

``` text
meta.EntityConfig
```

For SalesOrder, the configuration includes:

``` text
Entity       = SalesOrder
Type         = Fact
Staging      = stg.SalesOrder
Target       = dw.FactSales
Load Type    = Incremental
Business Key = OrderId + LineNumber
Watermark    = OrderDate
```

------------------------------------------------------------------------

# 7. GetFiles

Gets the source files from ADLS.

Example:

``` text
SalesOrder.csv
```

------------------------------------------------------------------------

# 8. Get_Watermark

Reads the previous watermark from:

``` text
meta.Watermark
```

Query:

``` sql
SELECT WatermarkValue
FROM meta.Watermark
WHERE EntityId = 4;
```

Example:

``` text
Watermark = 2026-08-25
```

### Purpose

> It tells the pipeline where the previous processing stopped.

------------------------------------------------------------------------

# 9. ForEachFile

Processes each file one by one.

Inside ForEach:

``` text
Copy_Current_File_To_Stage
          ↓
ProcessSalesOrder
```

------------------------------------------------------------------------

# 10. Copy to Staging

The source file is copied:

``` text
ADLS
 ↓
stg.SalesOrder
```

Staging is used before the final warehouse load.

------------------------------------------------------------------------

# 11. ProcessSalesOrder

Stored procedure:

``` text
dw.usp_ProcessSalesOrder
```

It is the main SalesOrder processing procedure.

It uses the watermark for incremental processing and handles the
SalesOrder processing logic, including duplicate handling where
implemented.

SalesOrder business key:

``` text
OrderId + LineNumber
```

Example:

``` text
OrderId   LineNumber
1001      1
1001      1
```

Same business key means the records are duplicates.

------------------------------------------------------------------------

# 12. Update_Watermark

After successful processing, the watermark is updated in:

``` text
meta.Watermark
```

Example:

``` text
Old Watermark = 2026-08-25

New record:
OrderDate = 2026-08-26

New Watermark = 2026-08-26
```

### Simple meaning

``` text
Get_Watermark
= Read where we stopped

Update_Watermark
= Save where we finished
```

This supports incremental processing.

------------------------------------------------------------------------

# 13. Reporting Layer

After the data is loaded into `dw.FactSales`, I use simple SQL queries
for reporting.

## Total Revenue

``` sql
SELECT SUM(NetAmount) AS TotalRevenue
FROM dw.FactSales;
```

Purpose:

> Get total sales revenue.

## Revenue by Product

``` sql
SELECT
    p.ProductName,
    SUM(f.NetAmount) AS Revenue
FROM dw.FactSales f
JOIN dw.DimProduct p
    ON f.ProductKey = p.ProductKey
GROUP BY p.ProductName
ORDER BY Revenue DESC;
```

Purpose:

> See revenue for each product.

## Revenue by Category

``` sql
SELECT
    p.Category,
    SUM(f.NetAmount) AS Revenue
FROM dw.FactSales f
JOIN dw.DimProduct p
    ON f.ProductKey = p.ProductKey
GROUP BY p.Category
ORDER BY Revenue DESC;
```

Purpose:

> See revenue for each category.

## Revenue by Date

``` sql
SELECT
    d.FullDate,
    SUM(f.NetAmount) AS Revenue
FROM dw.FactSales f
JOIN dw.DimDate d
    ON f.DateKey = d.DateKey
GROUP BY d.FullDate
ORDER BY d.FullDate;
```

Purpose:

> See daily revenue.

## Revenue by Customer

``` sql
SELECT
    c.CustomerName,
    SUM(f.NetAmount) AS Revenue
FROM dw.FactSales f
JOIN dw.DimCustomer c
    ON f.CustomerKey = c.CustomerKey
GROUP BY c.CustomerName
ORDER BY Revenue DESC;
```

Purpose:

> See revenue by customer.

## Revenue by Store

``` sql
SELECT
    s.StoreName,
    SUM(f.NetAmount) AS Revenue
FROM dw.FactSales f
JOIN dw.DimStore s
    ON f.StoreKey = s.StoreKey
GROUP BY s.StoreName
ORDER BY Revenue DESC;
```

Purpose:

> See revenue by store.

## Top 10 Products

``` sql
SELECT TOP 10
    p.ProductName,
    SUM(f.NetAmount) AS Revenue
FROM dw.FactSales f
JOIN dw.DimProduct p
    ON f.ProductKey = p.ProductKey
GROUP BY p.ProductName
ORDER BY Revenue DESC;
```

Purpose:

> Find the top 10 products by revenue.

------------------------------------------------------------------------

# 14. Overall Flow

``` text
Source File
    ↓
Master Pipeline
    ↓
Get Active Entities
    ↓
ForEach Entity
    ↓
Child Pipeline
    ↓
GetEntityConfig
    ↓
GetFiles
    ↓
Get_Watermark
    ↓
ForEachFile
    ↓
Copy to Staging
    ↓
ProcessSalesOrder
    ↓
Update_Watermark
    ↓
Data Warehouse
    ↓
Reporting Queries
```

------------------------------------------------------------------------

# 15. Simple Demo Explanation

> I created a metadata-driven Sales pipeline. I have four entities:
> Product, Customer, Store and SalesOrder. The Master pipeline reads the
> active entities from the metadata table and passes the EntityId to the
> reusable Child pipeline.
>
> The Child pipeline gets the entity configuration, gets the source
> files, reads the previous watermark and processes each file using
> ForEach. The source file is copied to staging and the SalesOrder
> stored procedure processes the data, including the implemented
> duplicate handling. After successful processing, the watermark is
> updated.
>
> Finally, the data is available in the warehouse, and I use simple
> reporting queries to get total revenue and revenue by product,
> category, date, customer and store.
