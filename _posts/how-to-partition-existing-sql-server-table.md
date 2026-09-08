# How to Partition an Existing SQL Server Table

Partitioning a large SQL Server table can make data lifecycle management easier and, for the right queries, enable **partition elimination**. The important point is that partitioning does not create several tables from the application's perspective: users still query one table, while SQL Server stores its rows in multiple internal partitions.

> This article adapts the approach demonstrated in the MSSQLTips article **“How to Partition an Existing SQL Server Table”** and expands it with a practical `Sales` example.

## Partitioning architecture

![SQL Server partitioning architecture](images/partition-architecture.svg)

The pieces are:

1. **Partitioning column** — the column whose value determines where each row belongs, such as `SaleDate`.
2. **Partition function** — defines the boundary values and determines the partition number.
3. **Partition scheme** — maps the partitions created by the function to filegroups.
4. **Partitioned index/table** — the clustered index (or heap) is placed on the partition scheme.

## Example: an existing Sales table

Assume this table already exists:

```sql
CREATE TABLE dbo.Sales
(
    SaleId      bigint NOT NULL,
    SaleDate    datetime2 NOT NULL,
    CustomerId  int NOT NULL,
    Amount      decimal(12,2) NOT NULL
);

ALTER TABLE dbo.Sales
ADD CONSTRAINT PK_Sales
PRIMARY KEY CLUSTERED (SaleId);
```

Initially, `SaleId` is the clustered primary key and the table is not partitioned.

## Step 1: choose the partitioning column

For a sales table, `SaleDate` is often a natural choice because retention, reporting, archival, and maintenance commonly follow date ranges.

In this example we want four ranges:

| Partition | Date range |
|---|---|
| P1 | Before 2024-01-01 |
| P2 | 2024 |
| P3 | 2025 |
| P4 | 2026-01-01 and later |

## Step 2: create the partition function

```sql
CREATE PARTITION FUNCTION SalesDatePF (datetime2)
AS RANGE RIGHT
FOR VALUES
(
    '2024-01-01',
    '2025-01-01',
    '2026-01-01'
);
```

Three boundary values create four partitions. With `RANGE RIGHT`, each boundary belongs to the partition on its right. Therefore `2025-01-01` is the first value in the 2025 partition.

You can test the mapping with `$PARTITION`:

```sql
SELECT $PARTITION.SalesDatePF('2025-01-01') AS PartitionNumber;
```

## Step 3: create the partition scheme

A simple configuration can place every partition on `PRIMARY`:

```sql
CREATE PARTITION SCHEME SalesDatePS
AS PARTITION SalesDatePF
ALL TO ([PRIMARY]);
```

Multiple filegroups are not required merely to partition a table. They are useful when your storage, backup/restore, or operational design calls for them.

## Step 4: handle the existing clustered primary key

The original table has:

```sql
PRIMARY KEY CLUSTERED (SaleId)
```

But this design will use `SaleDate` for the partitioned clustered index. One approach, like the one demonstrated in the source article, is to make the primary key nonclustered:

```sql
ALTER TABLE dbo.Sales
DROP CONSTRAINT PK_Sales;

ALTER TABLE dbo.Sales
ADD CONSTRAINT PK_Sales
PRIMARY KEY NONCLUSTERED (SaleId);
```

The primary-key constraint still exists; its backing index is simply no longer the clustered index.

## Step 5: create the partitioned clustered index

```sql
CREATE CLUSTERED INDEX CX_Sales_SaleDate
ON dbo.Sales(SaleDate)
ON SalesDatePS(SaleDate);
```

The important clause is:

```sql
ON SalesDatePS(SaleDate)
```

It places the clustered index on the partition scheme using `SaleDate` as the partitioning column.

![Before and after partitioning](images/before-after.svg)

Existing rows are redistributed during the clustered-index build. For example:

| SaleId | SaleDate | Destination |
|---:|---|---|
| 100 | 2023-06-10 | P1 |
| 101 | 2024-02-20 | P2 |
| 102 | 2024-12-01 | P2 |
| 103 | 2025-03-15 | P3 |
| 104 | 2026-07-01 | P4 |

## Verify the result

Use `sys.partitions` to inspect the physical partitions:

```sql
SELECT
    OBJECT_NAME(p.object_id) AS TableName,
    i.name AS IndexName,
    p.partition_number,
    p.rows
FROM sys.partitions AS p
JOIN sys.indexes AS i
  ON p.object_id = i.object_id
 AND p.index_id  = i.index_id
WHERE p.object_id = OBJECT_ID('dbo.Sales')
ORDER BY i.index_id, p.partition_number;
```

You can also count rows by partition:

```sql
SELECT
    $PARTITION.SalesDatePF(SaleDate) AS PartitionNumber,
    COUNT(*) AS RowCount
FROM dbo.Sales
GROUP BY $PARTITION.SalesDatePF(SaleDate)
ORDER BY PartitionNumber;
```

## Partition elimination

Partitioning does **not** automatically make every query faster. One important performance benefit is partition elimination.

```sql
SELECT SUM(Amount)
FROM dbo.Sales
WHERE SaleDate >= '2025-01-01'
  AND SaleDate <  '2026-01-01';
```

Because the predicate targets the partitioning key and corresponds to the 2025 range, SQL Server may be able to avoid reading unrelated partitions.

![Partition elimination example](images/partition-elimination.svg)

A query such as the following does not restrict `SaleDate`, so partition elimination may not help:

```sql
SELECT SUM(Amount)
FROM dbo.Sales
WHERE CustomerId = 10421;
```

Always verify actual behavior with the execution plan and runtime statistics rather than assuming partitioning will improve a workload.

## What about nonclustered indexes?

Existing nonclustered indexes do not automatically become aligned merely because the clustered structure is partitioned.

For example:

```sql
CREATE INDEX IX_Sales_Customer
ON dbo.Sales(CustomerId);
```

may remain non-aligned. If your design requires an aligned index, recreate it on the partition scheme while satisfying SQL Server's index-key requirements. For example:

```sql
CREATE INDEX IX_Sales_Customer
ON dbo.Sales(CustomerId, SaleDate)
ON SalesDatePS(SaleDate);
```

Index alignment is particularly important when designing for partition switching.

## Sliding-window partitioning

A common production design keeps recent periods online while retiring old periods and preparing a new range.

![Sliding-window partition strategy](images/sliding-window.svg)

SQL Server provides:

```sql
ALTER PARTITION FUNCTION ...
SPLIT RANGE (...);
```

to add a boundary, and:

```sql
ALTER PARTITION FUNCTION ...
MERGE RANGE (...);
```

to remove one.

Combined with partition switching, this can support efficient data-retention and archival workflows.

## RANGE RIGHT versus RANGE LEFT

With:

```sql
CREATE PARTITION FUNCTION PF (date)
AS RANGE RIGHT
FOR VALUES ('2025-01-01');
```

the boundary belongs to the right partition:

```text
P1: date < 2025-01-01
P2: date >= 2025-01-01
```

With `RANGE LEFT`, the boundary belongs to the left partition:

```text
P1: date <= 2025-01-01
P2: date > 2025-01-01
```

For calendar-based designs, `RANGE RIGHT` is often intuitive because a boundary such as `2026-01-01` represents the beginning of the new period.

## Production considerations

Rebuilding a large existing table onto a partition scheme is not a trivial metadata change. Building the clustered index can involve reading and rewriting large amounts of data, sorting, transaction-log growth, I/O, blocking, and significant temporary/storage requirements.

Before partitioning a production table, evaluate:

- Table and index sizes
- Primary-key and clustered-index design
- Foreign keys and dependencies
- Existing nonclustered indexes
- Transaction-log capacity and recovery model
- Free disk and temporary space
- Maintenance-window requirements
- Online index-operation support for your SQL Server version/edition
- Whether indexes must be aligned
- Whether you intend to use partition switching
- Backup, restore, retention, and archival requirements

## Summary

The conversion can be visualized as:

```text
BEFORE

dbo.Sales
   |
Clustered PK (SaleId)
   |
One partition


AFTER

dbo.Sales
   |
Clustered index (SaleDate)
   |
Partition scheme
   |
P1 | P2 | P3 | P4

PK_Sales remains a nonclustered primary key on SaleId.
```

The overall process is:

1. Choose a suitable partitioning column.
2. Create the partition function.
3. Create the partition scheme.
4. Modify the existing clustered-index/primary-key design if necessary.
5. Build the clustered index on the partition scheme.
6. Verify the partitions.
7. Review and align other indexes when required.
8. Test query plans and operational procedures before production deployment.

## References

- MSSQLTips — *How to Partition an Existing SQL Server Table*: https://www.mssqltips.com/sqlservertip/2888/how-to-partition-an-existing-sql-server-table/
- Microsoft Learn — *Create partitioned tables and indexes*: https://learn.microsoft.com/en-us/sql/relational-databases/partitions/create-partitioned-tables-and-indexes
- Microsoft Learn — *$PARTITION (Transact-SQL)*: https://learn.microsoft.com/en-us/sql/t-sql/functions/partition-transact-sql
- Microsoft Learn — *ALTER PARTITION FUNCTION (Transact-SQL)*: https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-partition-function-transact-sql
