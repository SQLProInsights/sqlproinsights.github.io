---
layout: post
title: "Troubleshooting High CPU Usage in SQL Server"
date: 2023-10-13

description: "Troubleshooting High CPU Usage in SQL Server"

categories:
  - Performance

tags:
  - Database
  - SQL Server
  - Database Administration
  - Cloud

image: /assets/images/social-preview.png
---

High CPU utilization is one of the most common performance problems SQL Server administrators encounter. When CPU usage remains consistently high, users may experience slow queries, application timeouts, blocking, and overall degradation in database performance.

However, high CPU usage does not necessarily mean that the server needs more CPU. In many cases, the underlying cause is inefficient queries, missing indexes, outdated statistics, excessive compilations, or an inappropriate execution plan.

This article provides a practical, step-by-step approach for troubleshooting high CPU usage in SQL Server.

## 1. Confirm SQL Server Is Actually Using the CPU
Before troubleshooting SQL Server, verify that the SQL Server process is responsible for the CPU utilization.

At the operating system level, check:
  - Task Manager
  - Resource Monitor
  - Performance Monitor
  - Process Explorer

Look for the `sqlservr.exe` process.

If overall server CPU utilization is 90% but SQL Server is only consuming 30%, another application or operating system process may be responsible.

If SQL Server is consuming most of the available CPU, continue troubleshooting inside SQL Server.

## 2. Check Current SQL Server CPU Utilization

SQL Server maintains CPU utilization information in the ring buffers.

```sql
DECLARE @ts_now BIGINT =
(
    SELECT cpu_ticks / (cpu_ticks / ms_ticks)
    FROM sys.dm_os_sys_info
);

SELECT TOP (30)
       DATEADD(ms, -1 * (@ts_now - [timestamp]), GETDATE()) AS EventTime,
       SQLProcessUtilization AS SQLServerCPU,
       SystemIdle AS SystemIdleCPU,
       100 - SystemIdle - SQLProcessUtilization AS OtherProcessCPU
FROM
(
    SELECT record.value('(./Record/@id)[1]', 'int') AS record_id,
           record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
           record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS SQLProcessUtilization,
           [timestamp]
    FROM
    (
        SELECT [timestamp],
               CONVERT(XML, record) AS record
        FROM sys.dm_os_ring_buffers
        WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
          AND record LIKE '%<SystemHealth>%'
    ) AS x
) AS y
ORDER BY record_id DESC;

```

This helps distinguish CPU consumption between:

- SQL Server
- Other processes
- Idle CPU

## 3. Find Queries Currently Consuming CPU

If the CPU problem is happening right now, identify active requests consuming CPU.

```sql
SELECT 
	r.session_id, 
	r.status, 
	r.cpu_time, 
	r.total_elapsed_time, 
	r.logical_reads, 
	r.reads, 
	r.writes, 
	r.wait_type, 
	r.wait_time, 
	DB_NAME(r.database_id) AS DatabaseName, 
	s.host_name, s.program_name, s.login_name, 
	SUBSTRING
		( t.text, 
		(r.statement_start_offset / 2) + 1, 
		( 
			( 
				CASE r.statement_end_offset 
					WHEN -1 THEN DATALENGTH(t.text) 
					ELSE r.statement_end_offset 
				END 
				- r.statement_start_offset 
			) / 2 
		) + 1 
	    ) AS RunningStatement 
	FROM sys.dm_exec_requests AS r 
	INNER JOIN sys.dm_exec_sessions AS s 
		ON r.session_id = s.session_id 
	CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t 
	WHERE r.session_id <> @@SPID 
	ORDER BY r.cpu_time DESC;

Pay particular attention to queries with:

 - High cpu_time
 - High logical reads
 - Long elapsed time
 - Large scans
 - Parallel execution

A query performing millions of logical reads may consume significant CPU even when the storage subsystem is performing well.

## 4. Find the Highest CPU Queries in the Plan Cache

Sometimes the CPU spike has already passed. In that case, query the plan cache to identify historically expensive statements.

```sql
SELECT TOP (20) 
	qs.total_worker_time / 1000.0 AS TotalCPU_ms, 
	qs.execution_count, (qs.total_worker_time / NULLIF
	(qs.execution_count, 0)) / 1000.0 AS AvgCPU_ms, 
	qs.total_elapsed_time / 1000.0 AS TotalElapsed_ms, 
	qs.total_logical_reads, 
	qs.last_execution_time, 
	DB_NAME(st.dbid) AS DatabaseName, 
	SUBSTRING ( 
		st.text, 
		(qs.statement_start_offset / 2) + 1, 
		( 
		  ( 
		     CASE qs.statement_end_offset 
			WHEN -1 THEN DATALENGTH(st.text) 
			ELSE qs.statement_end_offset 
		     END 
		     - qs.statement_start_offset 
		) / 2 
	      ) + 1 ) 
	   AS QueryText, 
	   qp.query_plan 
	FROM sys.dm_exec_query_stats AS qs 
	CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st 
	CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp 
	ORDER BY qs.total_worker_time DESC;
```
`total\\\_worker\\\_time` represents CPU time consumed by the query.

Do not look only at total CPU. Also compare:

`Total CPU`  
`Average CPU per execution`  
`Execution count`  
`Logical reads`  
`Last execution time`  

A query consuming 500 ms of CPU but executing 100,000 times may have a greater impact than a query consuming 20 seconds once per day.

**Note:** Plan-cache DMVs contain cumulative statistics only while the plan remains cached. SQL Server restart, plan eviction, recompilation, and cache clearing can reset or remove this information.

## 5. Check Query Store

For databases where Query Store is enabled, it is often one of the best places to investigate historical CPU problems.

First check whether Query Store is enabled:

```sql
SELECT
    name,
    is_query_store_on
FROM sys.databases;
```
You can also review the configuration:

```sql
SELECT * FROM sys.database_query_store_options;
```
Query Store can help identify:

 - Queries with high CPU consumption
 - Execution plan changes
 - Performance regressions
 - Queries that became slower after deployment
 - Differences between historical and current execution plans

This is particularly useful when someone reports:

**"CPU was 95% at 2:00 AM, but everything looks normal now."**

DMVs may no longer contain enough information, while Query Store can provide historical execution statistics.

## 6. Examine the Execution Plan

After identifying a high-CPU query, review its actual execution plan.

Look for operators such as:

 - Table Scan
 - Clustered Index Scan
 - Large Index Scan
 - Sort
 - Hash Match
 - Nested Loops processing excessive rows
 - Key Lookup executed thousands or millions of times
 - Spools
 - Excessive parallelism

Also compare:

`Estimated Number of Rows`  
`vs.`  
`Actual Number of Rows`

Large differences can indicate cardinality estimation problems, stale statistics, parameter sensitivity, or data distribution issues.

For example:

`Estimated Rows: 100`  
`Actual Rows:    5,000,000`

SQL Server may have selected an execution strategy appropriate for 100 rows but extremely inefficient for five million rows.

## 7. Check for Missing or Inefficient Indexes

Missing indexes are a common cause of unnecessary CPU consumption.

Without an appropriate index, SQL Server may scan millions of rows to return only a few records.

You can review missing-index DMVs:

```sql
SELECT TOP (20)
    DB_NAME(mid.database_id) AS DatabaseName,
    OBJECT_NAME(mid.object_id, mid.database_id) AS TableName,
    migs.user_seeks,
    migs.avg_total_user_cost,
    migs.avg_user_impact,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_group_stats AS migs
INNER JOIN sys.dm_db_missing_index_groups AS mig
    ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details AS mid
    ON mig.index_handle = mid.index_handle
ORDER BY
    migs.avg_total_user_cost *
    migs.avg_user_impact *
    (migs.user_seeks + migs.user_scans) DESC;
```

## Important

Do not automatically create every index recommended by SQL Server.

Before adding an index, consider:

 - Existing indexes
 - Index overlap
 - Write overhead
 - Index size
 - Included columns
 - Overall workload

Missing-index DMVs should be treated as troubleshooting clues, not automatic index-creation instructions.

## 8. Check Statistics

Outdated statistics can cause poor cardinality estimates and inefficient execution plans.

Check statistics for a table:

```sql
SELECT 
	OBJECT_NAME(s.object_id) AS TableName, 
	s.name AS StatisticsName, 
	STATS_DATE(s.object_id, s.stats_id) AS StatisticsLastUpdated 
FROM sys.stats AS s 
WHERE s.object_id = OBJECT_ID('dbo.YourTable') 
ORDER BY StatisticsLastUpdated;
```
If statistics are outdated and the workload justifies it, update the relevant statistics:

```sql
UPDATE STATISTICS dbo.YourTable;
```

Or target a specific statistics object:

```sql
UPDATE STATISTICS dbo.YourTable YourStatisticsName;
```

Avoid blindly running full-scan statistics updates across a large production environment during peak hours. Determine which objects are contributing to the problematic execution plans first.

## 9. Check for Parameter-Sensitive Queries

A stored procedure may perform well for one parameter value but poorly for another.

For example:

```sql
EXEC dbo.GetOrders @CustomerID = 100;
```
may return 10 rows, while:

```sql
EXEC dbo.GetOrders @CustomerID = 5000;
```
may return 2 million rows.

If the same cached execution plan is inappropriate for both workloads, CPU consumption can increase dramatically.

Possible approaches include:

 - Query tuning
 - Better indexing
 - OPTION (RECOMPILE) where appropriate
 - OPTIMIZE FOR
 - Dynamic SQL
 - Query Store plan management
 - Parameter Sensitive Plan optimization on supported SQL Server versions

Do not immediately assume every parameter-related performance problem should be solved with OPTION (RECOMPILE). Recompilation itself has CPU cost.

## 10. Check for Excessive Compilations and Recompilations

Compilation requires CPU.

Useful Performance Monitor counters include:

`SQLServer:SQL Statistics`  
    `SQL Compilations/sec`  
    `SQL Re-Compilations/sec`  
    `Batch Requests/sec`

The ratio between compilations and batch requests can help determine whether excessive compilation is contributing to CPU pressure.

Common causes include:

 - Excessive ad hoc SQL
 - Frequent recompilation
 - Schema changes
 - Statistics changes
 - Poor application query design
 - Queries containing many different literal values

## 11. Check Ad Hoc Plan Cache Usage

Applications that generate large numbers of unique ad hoc queries can create plan-cache pressure and additional compilation overhead.

Check the plan cache:

```sql
SELECT 
	objtype, 
	cacheobjtype, 
	COUNT(*) AS PlanCount, 
	SUM(CAST(size_in_bytes AS BIGINT)) / 1024.0 / 1024.0 AS CacheSizeMB 
FROM sys.dm_exec_cached_plans 
GROUP BY 
	objtype, 
	cacheobjtype 
ORDER BY CacheSizeMB DESC;
```
You can also check whether optimize for ad hoc workloads is enabled:

```sql
EXEC sys.sp\\\_configure 'optimize for ad hoc workloads';
```

This option can be useful in environments containing large numbers of single-use ad hoc plans, but the workload should be evaluated before changing the server configuration.

## 12. Investigate Parallelism

Parallel queries can legitimately use multiple CPU cores.

Check current parallel requests:

```sql
SELECT 
	session_id, 
	request_id, 
	status, 
	command, 
	cpu_time, 
	total_elapsed_time, 
	wait_type, 
	dop, 
	parallel_worker_count 
FROM sys.dm_exec_requests 
WHERE dop > 1 
ORDER BY cpu_time DESC;
```

Also review SQL Server parallelism settings:

```sql
EXEC sys.sp\\\_configure 'max degree of parallelism';
EXEC sys.sp\\\_configure 'cost threshold for parallelism';
```

Parallelism is not automatically a problem.

A common mistake is reducing MAXDOP simply because CPU utilization is high. First identify which queries are consuming CPU and why.

## 13. Review CPU-Related Wait Statistics

Wait statistics provide another perspective on what SQL Server is experiencing.

```sql
SELECT TOP (30) 
	wait_type, 
	waiting_tasks_count, 
	wait_time_ms, 
	signal_wait_time_ms, 
	wait_time_ms - signal_wait_time_ms AS resource_wait_time_ms 
FROM sys.dm_os_wait_stats 
WHERE wait_type NOT LIKE 'SLEEP%' 
ORDER BY wait_time_ms DESC;
```

For CPU troubleshooting, pay attention to:

`SOS\\\_SCHEDULER\\\_YIELD`  
`CXPACKET`  
`CXCONSUMER`

`SOS\\\_SCHEDULER\\\_YIELD` can become significant when workers repeatedly yield the scheduler while performing CPU-intensive work.

However, wait statistics must be interpreted in the context of the workload. The existence of `SOS\\\_SCHEDULER\\\_YIELD` or `CXPACKET` does not by itself prove there is a CPU problem.

## 14. Check Scheduler Pressure

SQL Server scheduler information can help determine whether runnable tasks are waiting for CPU.

```sql
SELECT 
	scheduler_id, cpu_id, 
	status, 
	current_tasks_count, 
	runnable_tasks_count, 
	active_workers_count, 
	work_queue_count, 
	pending_disk_io_count 
FROM sys.dm_os_schedulers 
WHERE status = 'VISIBLE ONLINE';
```

Pay particular attention to:

`runnable\\\_tasks\\\_count`

A sustained runnable queue across multiple schedulers can indicate CPU pressure.

A single snapshot is not enough to make that determination. Capture several samples during the performance problem.

## 15. Check for Scalar Functions and CPU-Intensive Expressions

CPU consumption can also come from query logic rather than indexing.

Examples include:

```sql
WHERE CONVERT(VARCHAR(10), OrderDate, 112) = '20260820'
```

instead of a searchable range predicate such as:

```sql
WHERE OrderDate >= '20260820' 
 AND OrderDate < '20260821'
```

Other potential CPU-heavy patterns include:

 - Functions applied to indexed columns
 - Complex calculations
 - String manipulation
 - XML processing
 - JSON processing
 - Large sorts
 - Regular expression-like string operations
 - Row-by-row processing

Whenever possible, write predicates that allow SQL Server to efficiently seek into indexes.

## 16. Look for Application Workload Changes

Sometimes SQL Server has not changed at all.

Ask:

 - Was a new application version deployed?
 - Did traffic suddenly increase?
 - Did a scheduled job start?
 - Was a report introduced?
 - Did an ETL process overlap with production workload?
 - Did query execution frequency increase?
 - Did a database deployment change indexes or statistics?
 - Did an execution plan change?

A query that normally executes 100 times per hour may suddenly execute 100,000 times per hour.

Even an individually efficient query can become a major CPU consumer at sufficiently high execution frequency.

## 17. A Practical High-CPU Troubleshooting Workflow

When receiving a production alert such as:

SQL Server CPU = 95%

use a structured approach:

```text
High CPU Alert
      |
      v
Is sqlservr.exe consuming CPU?
      |
      +-- No --> Investigate OS / other processes
      |
      +-- Yes
            |
            v
Identify currently expensive queries
            |
            v
Check Query Store / historical CPU queries
            |
            v
Review execution plans
            |
            v
Check logical reads and row estimates
            |
            v
Check indexes and statistics
            |
            v
Check parameter-sensitive plans
            |
            v
Check compilation / recompilation
            |
            v
Review parallelism
            |
            v
Review waits and scheduler pressure
            |
            v
Tune the workload
            |
            v
Monitor and validate
```
## 18. What Not to Do During a High-CPU Incident

Avoid immediately taking actions such as:

```sql
DBCC FREEPROCCACHE;
```

or restarting SQL Server.

These actions may temporarily make the symptom disappear, but they also destroy valuable diagnostic information and can introduce additional compilation overhead.

Similarly, avoid immediately:

 - Increasing CPU resources
 - Changing MAXDOP
 - Changing cost threshold for parallelism
 - Rebuilding every index
 - Updating every statistic
 - Killing sessions without investigation

First collect evidence.

Then determine the root cause.

## 19. Useful Information to Capture During the Incident

CPU problems are much easier to troubleshoot when diagnostic information is captured while the problem is occurring.

Capture:

 - CPU utilization
 - Active requests
 - Query text
 - Execution plans
 - Query Store statistics
 - Wait statistics
 - Scheduler information
 - Logical reads
 - Execution counts
 - Blocking information
 - SQL Agent jobs running at the time
 - Application deployment history

If the problem occurs regularly, consider using Query Store, Extended Events, or a monitoring platform to preserve historical information.

## 20. Example Troubleshooting Scenario

Suppose SQL Server CPU suddenly reaches 95%.

You identify the highest CPU query and discover:

```sql
Execution Count:       45,000
Average CPU:           120 ms
Average Logical Reads: 185,000
```

The execution plan shows a clustered index scan against a 50-million-row table.

The query is:

```sql
SELECT 
	OrderID, 
	CustomerID, 
	OrderDate 
FROM dbo.Orders 
WHERE CustomerID = @CustomerID;
```
The table does not have an appropriate index on `CustomerID`.

After workload analysis, an appropriate index might be:

```sql
CREATE INDEX IX_Orders_CustomerID 
 ON dbo.Orders (CustomerID) 
INCLUDE (OrderID, OrderDate);
```

After testing, the execution plan changes from a large scan to an efficient seek and logical reads drop significantly.

In this situation, adding more CPU would only have masked the underlying problem. The root cause was inefficient data access.

**Conclusion**

High CPU utilization in SQL Server should be treated as a symptom, not automatically as the root cause.

A good troubleshooting process starts by answering three questions:

**Is SQL Server actually consuming the CPU?**  
**Which workload is consuming the CPU?**  
**Why is that workload requiring so much CPU?**

In many environments, the root cause eventually comes down to one or more of the following:

 - Inefficient queries
 - Excessive logical reads
 - Missing or poorly designed indexes
 - Outdated statistics
 - Poor execution plans
 - Parameter-sensitive queries
 - Excessive compilation
 - CPU-intensive query logic
 - Inappropriate parallelism
 - Increased application workload

The most important rule during a production CPU incident is simple:

**Collect evidence before making changes.**

Finding and tuning the workload responsible for CPU consumption usually provides a much better long-term solution than simply adding hardware or restarting SQL Server.
