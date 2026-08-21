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

Task Manager
Resource Monitor
Performance Monitor
Process Explorer

Look for the `sqlservr.exe` process.

If overall server CPU utilization is 90% but SQL Server is only consuming 30%, another application or operating system process may be responsible.

If SQL Server is consuming most of the available CPU, continue troubleshooting inside SQL Server.

## 2. Check Current SQL Server CPU Utilization

SQL Server maintains CPU utilization information in the ring buffers.

```sql
DECLARE @ts_now BIGINT = ( SELECT cpu_ticks / (cpu_ticks / ms_ticks) FROM sys.dm_os_sys_info ); SELECT TOP (30) DATEADD(ms, -1 * (@ts_now - [timestamp]), GETDATE()) AS EventTime, SQLProcessUtilization AS SQLServerCPU, SystemIdle AS SystemIdleCPU, 100 - SystemIdle - SQLProcessUtilization AS OtherProcessCPU FROM ( SELECT record.value('(./Record/@id)[1]', 'int') AS record_id, record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle, record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS SQLProcessUtilization, [timestamp] FROM ( SELECT [timestamp], CONVERT(XML, record) AS record FROM sys.dm_os_ring_buffers WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR' AND record LIKE '%<SystemHealth>%' ) AS x ) AS y ORDER BY record_id DESC;
```

This helps distinguish CPU consumption between:

SQL Server
Other processes
Idle CPU

## 3. Find Queries Currently Consuming CPU

If the CPU problem is happening right now, identify active requests consuming CPU.

```sql
SELECT r.session_id, r.status, r.cpu_time, r.total_elapsed_time, r.logical_reads, r.reads, r.writes, r.wait_type, r.wait_time, DB_NAME(r.database_id) AS DatabaseName, s.host_name, s.program_name, s.login_name, SUBSTRING( t.text, (r.statement_start_offset / 2) + 1, ( ( CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(t.text) ELSE r.statement_end_offset END - r.statement_start_offset ) / 2 ) + 1 ) AS RunningStatement FROM sys.dm_exec_requests AS r INNER JOIN sys.dm_exec_sessions AS s ON r.session_id = s.session_id CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t WHERE r.session_id <> @@SPID ORDER BY r.cpu_time DESC;
```

Pay particular attention to queries with:

High cpu_time
High logical reads
Long elapsed time
Large scans
Parallel execution

A query performing millions of logical reads may consume significant CPU even when the storage subsystem is performing well.

## 4. Find the Highest CPU Queries in the Plan Cache

Sometimes the CPU spike has already passed. In that case, query the plan cache to identify historically expensive statements.

```sql
SELECT TOP (20) qs.total_worker_time / 1000.0 AS TotalCPU_ms, qs.execution_count, (qs.total_worker_time / NULLIF(qs.execution_count, 0)) / 1000.0 AS AvgCPU_ms, qs.total_elapsed_time / 1000.0 AS TotalElapsed_ms, qs.total_logical_reads, qs.last_execution_time, DB_NAME(st.dbid) AS DatabaseName, SUBSTRING( st.text, (qs.statement_start_offset / 2) + 1, ( ( CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text) ELSE qs.statement_end_offset END - qs.statement_start_offset ) / 2 ) + 1 ) AS QueryText, qp.query_plan FROM sys.dm_exec_query_stats AS qs CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) AS qp ORDER BY qs.total_worker_time DESC;
```
`total_worker_time` represents CPU time consumed by the query.

Do not look only at total CPU. Also compare:

`Total CPU
Average CPU per execution
Execution count
Logical reads
Last execution time'

A query consuming 500 ms of CPU but executing 100,000 times may have a greater impact than a query consuming 20 seconds once per day.
