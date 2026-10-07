---
title: "How SQL Server Index Column Order Impacts CPU and Query Performance"
date: 2023-10-11
categories:
  - T-SQL
  - SQL Server
  - Performance
 
tags:
  - SQL Server
  - T-SQL
  - Blocking
  - Troubleshooting
  - Performance
description: "Hands-On Lab: How SQL Server Index Column Order Impacts CPU and Query Performance."

image: /assets/images/social-preview.png
---

# How SQL Server Index Column Order Impacts CPU and Query Performance

A query can have an index and still perform poorly. One of the reasons is **index key column order**.

In this hands-on lab, we will build a small SQL Server workload that demonstrates how index design can affect:

- CPU usage
- Logical reads
- Sort operations
- Query execution time
- Index seek efficiency
- Overall query performance

Rather than simply discussing indexing theory, we will create the problem ourselves, measure it, change the index, and measure it again.
---
## What You Will Learn

By the end of this lab, you will understand:
1.Why index column order matters.
2.How equality and range predicates affect index usage.
3.How ORDER BY can influence index design.
4.How to identify unnecessary Sort operations.
5.How included columns can eliminate Key Lookups.
6.How to compare CPU and logical reads before and after an index change.
7.Why an index that looks reasonable on paper may not be the best index for a workload.

## Lab Scenario

Imagine an application that frequently retrieves the most recent orders for a specific customer.
The application runs a query similar to:
```sql
SELECT TOP (50)
       OrderID,
       CustomerID,
       OrderDate,
       OrderStatus,
       TotalAmount
FROM dbo.CustomerOrders
WHERE CustomerID = 1250
  AND OrderDate >= '2026-01-01'
ORDER BY OrderDate DESC;
```
The query has three important characteristics:
```text
CustomerID = 1250       → Equality predicate

OrderDate >= ...        → Range predicate

ORDER BY OrderDate DESC → Ordering requirement
```
Our goal is to design an index that allows SQL Server to perform this work efficiently.

# Step 1 — Create the Lab Database

Create a separate database for the lab.
```sql
CREATE DATABASE SQLProInsights_IndexLab;
GO

USE SQLProInsights_IndexLab;
GO
```
If you already have a development database, you can perform the lab there instead.

# Step 2 — Create the Test Table

Create a table representing customer orders.
```sql
CREATE TABLE dbo.CustomerOrders
(
    OrderID       BIGINT IDENTITY(1,1) NOT NULL,
    CustomerID    INT NOT NULL,
    OrderDate     DATETIME2(0) NOT NULL,
    OrderStatus   VARCHAR(20) NOT NULL,
    TotalAmount   DECIMAL(12,2) NOT NULL,
    ProductID     INT NOT NULL,
    SalesRegion   VARCHAR(20) NOT NULL,
    CreatedBy     VARCHAR(50) NOT NULL,

    CONSTRAINT PK_CustomerOrders
        PRIMARY KEY CLUSTERED (OrderID)
);
GO
```
The clustered index is on OrderID. This is intentional.

Our query will search by CustomerID and OrderDate, not by OrderID.

# Step 3 — Generate Test Data

For this lab, let's generate 1 million orders.
```sql
;WITH N AS
(
    SELECT TOP (1000000)
           ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
    FROM sys.all_objects a
    CROSS JOIN sys.all_objects b
)
INSERT INTO dbo.CustomerOrders
(
    CustomerID,
    OrderDate,
    OrderStatus,
    TotalAmount,
    ProductID,
    SalesRegion,
    CreatedBy
)
SELECT
    ((n - 1) % 10000) + 1 AS CustomerID,

    DATEADD
    (
        DAY,
        -(n % 1460),
        CAST('2026-12-31' AS DATETIME2)
    ) AS OrderDate,

    CASE n % 4
        WHEN 0 THEN 'Completed'
        WHEN 1 THEN 'Pending'
        WHEN 2 THEN 'Cancelled'
        ELSE 'Shipped'
    END AS OrderStatus,

    CAST
    (
        25.00 + ((n * 17) % 100000) / 100.0
        AS DECIMAL(12,2)
    ) AS TotalAmount,

    ((n - 1) % 5000) + 1 AS ProductID,

    CASE n % 4
        WHEN 0 THEN 'East'
        WHEN 1 THEN 'West'
        WHEN 2 THEN 'North'
        ELSE 'South'
    END AS SalesRegion,

    CONCAT('User', ((n - 1) % 100) + 1)
FROM N;
GO
```
Check the number of rows:
```sql
SELECT COUNT(*) AS TotalRows
FROM dbo.CustomerOrders;
GO
```
You should have approximately:
```text
1,000,000
```
rows.

# Step 4 — Update Statistics

Before beginning the performance tests, update the table statistics.
```sql
UPDATE STATISTICS dbo.CustomerOrders
WITH FULLSCAN;
GO
```
This gives the optimizer better information about the test data.

# Step 5 — Establish a Baseline

Before creating a non-clustered index, let's see how SQL Server handles the query.

Turn on:

 - Actual Execution Plan
 - Statistics IO
 - Statistics TIME

In SSMS:

**Query → Include Actual Execution Plan**

or press:
```text
Ctrl + M
```
Then execute:
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT TOP (50)
       OrderID,
       CustomerID,
       OrderDate,
       OrderStatus,
       TotalAmount
FROM dbo.CustomerOrders
WHERE CustomerID = 1250
  AND OrderDate >= '2026-01-01'
ORDER BY OrderDate DESC;

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```
Record the results.

We are interested in:
```text
CPU time
Elapsed time
Logical reads
Execution plan operators
```
Don't worry if your numbers are different from the examples in this article.

Hardware, SQL Server version, memory, database configuration, and workload will affect the results.

# Step 6 — Create Our First Index

Let's create an index that might initially seem reasonable:

```sql
CREATE INDEX IX_CustomerOrders_OrderDate_CustomerID
ON dbo.CustomerOrders
(
    OrderDate,
    CustomerID
);
GO
```
At first glance, this looks useful.

The query searches on both:
```text
OrderDate
CustomerID
```
So why not put them both into the index?

This is where column order becomes important.

# Step 7 — Run the Query Again

Execute the same query:
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT TOP (50)
       OrderID,
       CustomerID,
       OrderDate,
       OrderStatus,
       TotalAmount
FROM dbo.CustomerOrders
WHERE CustomerID = 1250
  AND OrderDate >= '2026-01-01'
ORDER BY OrderDate DESC;

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```
Now examine the execution plan.

Depending on the optimizer's choices and your test environment, you may see operators such as:
```text
Index Seek
Sort
Key Lookup
```
The exact plan is workload-dependent.

That is an important point:

`Never assume what SQL Server will do. Look at the actual execution plan.`

# Step 8 — Create an Alternative Index

Now let's change the key order.
```sql
CREATE INDEX IX_CustomerOrders_CustomerID_OrderDate
ON dbo.CustomerOrders
(
    CustomerID,
    OrderDate
);
GO
```
This index begins with:
```text
CustomerID
```
which is an equality predicate:
```sql
WHERE CustomerID = 1250
```
followed by:
```text
OrderDate
```
which is used for the date restriction and ordering.

# Step 9 — Test Again

Run the query again.
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT TOP (50)
       OrderID,
       CustomerID,
       OrderDate,
       OrderStatus,
       TotalAmount
FROM dbo.CustomerOrders
WHERE CustomerID = 1250
  AND OrderDate >= '2026-01-01'
ORDER BY OrderDate DESC;

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```
Compare the new execution plan with the previous one.

Pay particular attention to:
```text
Index Seek
Sort
Key Lookup
Logical Reads
CPU Time
Elapsed Time
```
# Step 10 — Why Did Column Order Matter?

Our first index was:
```text
(OrderDate, CustomerID)
```
The second was:
```text
(CustomerID, OrderDate)
```
They contain exactly the same two columns.

The difference is their order.

That difference can significantly change how SQL Server navigates the B-tree structure.

With:
```text
CustomerID → OrderDate
```
SQL Server can first narrow the search to the requested customer and then work within that customer's portion of the index using OrderDate.

Conceptually:
```text
CustomerID
   |
   +-- Customer 1250
          |
          +-- OrderDate
                 |
                 +-- Recent orders
```
This can provide a much more targeted access path for our query.

# Step 11 — But We Still Have a Potential Problem

Look at the columns returned by the query:
```text
OrderID
CustomerID
OrderDate
OrderStatus
TotalAmount
```
Our index contains:
```text
CustomerID
OrderDate
```
But it does not contain:
```text
OrderStatus
TotalAmount
```
or the clustered key:
```text
OrderID
```
Depending on the execution plan and the number of rows SQL Server needs to retrieve, it may perform additional lookups.

Let's inspect the plan.

If you see a:
```text
Key Lookup
```
operator, SQL Server is using the nonclustered index to find rows and then going back to the clustered index to retrieve additional columns.

A lookup isn't automatically bad.

The important question is:

`How many rows are being looked up?'

# Step 12 — Create a Covering Index

For this particular query, we can include the additional columns:
```sql
CREATE INDEX IX_CustomerOrders_CustomerID_OrderDate_Covering
ON dbo.CustomerOrders
(
    CustomerID,
    OrderDate
)
INCLUDE
(
    OrderStatus,
    TotalAmount
);
GO
```
Notice that we did not put every column into the key.

The key remains:
```text
CustomerID,
OrderDate
```
The additional columns are included as payload.

# Step 13 — Test the Covering Index

Run:
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT TOP (50)
       OrderID,
       CustomerID,
       OrderDate,
       OrderStatus,
       TotalAmount
FROM dbo.CustomerOrders
WHERE CustomerID = 1250
  AND OrderDate >= '2026-01-01'
ORDER BY OrderDate DESC;

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```
Inspect the execution plan again.

You may now find that SQL Server can obtain the requested data directly from the non-clustered index without needing a Key Lookup.

# Step 14 — Compare the Results

Create a simple test table to record your observations:

```text
Test   Index                                Logical Reads	        Sort?	        Key Lookup?
----   -----                                -------------	        -----	        -----------
1	No nonclustered index                 Record                Record        Record          
2	(OrderDate, CustomerID)                 Record		            Record	      Record
3	(CustomerID, OrderDate)                 Record		            Record	      Record
4	(CustomerID, OrderDate) + INCLUDE                 Record		            Record	      Record     
```

Your numbers will depend on your environment.

The important thing is to compare the relative change.

# Step 15 — Look at Logical Reads

STATISTICS IO reports logical reads.

For example, you may see something similar to:
```text
Table 'CustomerOrders'.
Scan count 1,
logical reads 1250
```
The exact number is not important for this demonstration.

What matters is whether the improved access path allows SQL Server to read fewer pages.

Reducing logical reads can reduce:
 - CPU work
 - memory pressure
 - buffer pool activity
 - I/O pressure

and can improve overall query scalability.

# Step 16 — Look at CPU Time

STATISTICS TIME reports CPU time and elapsed time.

For example:
```text
SQL Server Execution Times:
   CPU time = 45 ms,
   elapsed time = 52 ms.
```
After an index improvement, you might see lower values.

However, don't expect every index change to produce a dramatic CPU reduction.

The benefit depends on:
 - table size
 - data distribution
 - selectivity
 - query frequency
 - concurrent workload
 - hardware
 - SQL Server version
 - existing indexes
 - optimizer decisions

# Step 17 — Understand the Role of TOP

Our query contains:
```text
TOP (50)
```
This is important.

The application doesn't need every qualifying order.

It only needs the latest 50.

If SQL Server can efficiently navigate an index that provides the required order, it may be able to locate those rows without sorting a much larger set of data.

This is one reason that understanding the query's complete access pattern is more useful than simply indexing every WHERE column.

# Step 18 — Don't Automatically Remove Every Sort

You may notice a Sort operator in some execution plans and immediately think:

`"The Sort is bad."`

That isn't always true.

Sorting is a normal database operation.

The question is:

`Is the cost of the Sort significant for this workload?`

A Sort that processes 50 rows may be irrelevant.

A Sort that processes millions of rows thousands of times per hour is a different story.

Always evaluate the cost in context.

# Step 19 — Don't Automatically Create a Covering Index

Our covering index looks attractive:
```sql
CREATE INDEX IX_CustomerOrders_CustomerID_OrderDate_Covering
ON dbo.CustomerOrders
(
    CustomerID,
    OrderDate
)
INCLUDE
(
    OrderStatus,
    TotalAmount
);
```
But there is a trade-off.

Every additional index has a maintenance cost.

When rows are inserted or modified, SQL Server may need to maintain the index.

For a heavily updated table, excessive indexing can become a performance problem of its own.

Therefore:

`A covering index should solve a meaningful workload problem, not simply eliminate every Key Lookup you encounter.`

# Step 20 — Test the Index Under Concurrency

A DBA should also consider how the query behaves when multiple sessions execute it simultaneously.

A query that uses:
```text
10 ms CPU
```
once may look harmless.

But if the application executes it:
```text
1,000 times
```
the total CPU consumption can become significant.

Conceptually:
```text
10 ms × 1,000 executions
=
10,000 ms CPU
=
10 seconds of CPU
```
This is why frequently executed queries can create significant server CPU pressure even when individual executions appear fast.

# Step 21 — Find High-CPU Queries

Once you understand how index design can affect CPU, you can apply the same approach to production troubleshooting.

A useful starting point is:
```sql
SELECT TOP (20)
       qs.total_worker_time / 1000 AS total_cpu_ms,
       qs.execution_count,
       qs.total_worker_time /
           NULLIF(qs.execution_count, 0) / 1000 AS avg_cpu_ms,
       qs.total_logical_reads,
       qs.total_elapsed_time / 1000 AS total_elapsed_ms,
       DB_NAME(st.dbid) AS database_name,
       SUBSTRING
       (
           st.text,
           (qs.statement_start_offset / 2) + 1,
           (
               (
                   CASE
                       WHEN qs.statement_end_offset = -1
                       THEN DATALENGTH(st.text)
                       ELSE qs.statement_end_offset
                   END
                   - qs.statement_start_offset
               ) / 2
           ) + 1
       ) AS query_text
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
ORDER BY qs.total_worker_time DESC;
```
This can help identify statements that have accumulated significant CPU since their statistics were last reset.

For historical analysis, Query Store is generally more useful because it provides persisted query performance information.

# Step 22 — A Practical DBA Workflow

When I encounter a high-CPU SQL Server workload, I prefer to follow this process:
```text
High CPU
   |
   v
Identify expensive queries
   |
   v
Capture execution plan
   |
   v
Review CPU + logical reads
   |
   v
Check predicates
   |
   v
Review index design
   |
   v
Check Sort / Lookup / Scan operators
   |
   v
Design candidate index
   |
   v
Test
   |
   v
Compare before vs. after
   |
   v
Test concurrency
   |
   v
Deploy and monitor
```
This prevents the common mistake of creating indexes based only on intuition.

# Important Lessons From This Lab

## 1. Index column order matters

These are not equivalent from the optimizer's perspective:
```text
(CustomerID, OrderDate)
```
and:
```text
(OrderDate, CustomerID)
```
Even though they contain the same columns.

## 2. Equality and range predicates behave differently

Our query contains:
```text
CustomerID = 1250
```
and:
```text
OrderDate >= '2026-01-01'
```
The access pattern should be considered when determining index key order.

## 3. ORDER BY matters

An index can sometimes provide rows in the required order and reduce the need for an explicit Sort.

This can become particularly useful with:
```text
TOP
```
queries.

## 4. Key Lookups aren't automatically bad

A lookup may be perfectly reasonable for a small number of rows.

The problem is when a lookup becomes expensive because SQL Server has to perform a very large number of them.

## 5. Covering indexes have trade-offs

A covering index can reduce lookups, but it also increases index size and maintenance overhead.

## 6. Measure everything

Before and after an index change, compare:
```text
CPU time
Elapsed time
Logical reads
Execution plan
Rows processed
```
Don't rely on assumptions.

# Cleanup

Because this is a lab environment, you can remove the indexes after completing the exercise.
```sql
DROP INDEX IF EXISTS IX_CustomerOrders_OrderDate_CustomerID
ON dbo.CustomerOrders;

DROP INDEX IF EXISTS IX_CustomerOrders_CustomerID_OrderDate
ON dbo.CustomerOrders;

DROP INDEX IF EXISTS IX_CustomerOrders_CustomerID_OrderDate_Covering
ON dbo.CustomerOrders;
GO
```
If you created the database specifically for this lab, you can remove it as well:
```sql
USE master;
GO

DROP DATABASE SQLProInsights_IndexLab;
GO
```

# Final Takeaway

A common misconception is:

`"If a query has an index, SQL Server should be fast."`

That's not necessarily true.

The more useful question is:

`Does the index provide an efficient access path for the way the query actually retrieves and returns data?`

In our lab, the difference between:
```sql
(OrderDate, CustomerID)
```
and:
```sql
(CustomerID, OrderDate)
```
illustrates why index key order deserves careful consideration.

When you combine that understanding with execution-plan analysis, `STATISTICS IO, STATISTICS TIME`, Query Store, and realistic workload testing, indexing becomes a much more powerful performance-tuning tool.

The goal isn't to create more indexes.

The goal is to make SQL Server do less unnecessary work.

# SQLProInsights Performance Tuning Checklist

Before creating or changing an index, ask:
```text
☐ What query am I trying to improve?
☐ How frequently does it execute?
☐ What are the equality predicates?
☐ What are the range predicates?
☐ What columns are used for JOIN?
☐ What columns are used for ORDER BY?
☐ Is there a TOP/OFFSET requirement?
☐ Is SQL Server performing a Scan?
☐ Is there an expensive Sort?
☐ Is there a Key Lookup?
☐ How many logical reads occur?
☐ How much CPU does the query consume?
☐ Can the existing index be modified instead of adding another one?
☐ What will the new index cost for INSERT/UPDATE/DELETE?
☐ Did I measure before and after?
☐ Did I test with realistic workload/concurrency?
```

