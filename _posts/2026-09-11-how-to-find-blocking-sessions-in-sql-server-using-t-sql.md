---
title: "How to Find Blocking Sessions in SQL Server Using T-SQL"
date: 2021-02-11
categories:
  - T-SQL
tags:
  - SQL Server
  - T-SQL
  - Blocking
  - Troubleshooting
  - Performance
description: "Learn how to identify blocking sessions, blocking SPIDs, wait types, wait resources, SQL statements, and blocking chains in SQL Server using T-SQL."

image: /assets/images/social-preview.png
---

# How to Find Blocking Sessions in SQL Server Using T-SQL

Blocking is one of the most common performance problems SQL Server DBAs troubleshoot.

A blocked session occurs when one SQL Server session is waiting for another session to release a resource such as a row, page, key, or table lock.
Some blocking is normal. It becomes a problem when the blocking lasts long enough to affect application response time, batch processing, or other database activity.
In this article, we will use practical T-SQL queries to identify:

- Blocked sessions
- Blocking session IDs
- Wait types
- Wait duration
- Wait resources
- Database name
- Login and host information
- Running SQL statements
- Blocking chains
- The likely head blocker
---

## What Is Blocking in SQL Server?

Consider two SQL Server sessions:
```text
Session 51
   |
   | Holds a lock
   v
Database Resource
   ^
   | Waiting
   |
Session 72
```
Session 51 owns a resource required by Session 72.
SQL Server therefore makes Session 72 wait until Session 51 completes its transaction or releases the required lock.
Blocking can also form a chain:

```text
Session 51
    |
    v
Session 72
    |
    v
Session 85
```
In this example:
```text
51 = Head blocker
72 = Blocked by 51 and blocking 85
85 = Blocked by 72
```
Finding the head blocker is usually more important than simply identifying the last blocked session.
Basic Blocking Check

A quick way to identify blocked requests is to query `sys.dm_exec_requests`.
```sql
SELECT
    session_id,
    blocking_session_id,
    status,
    wait_type,
    wait_time,
    wait_resource
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
GO
```
If `blocking_session_id` contains a non-zero value, that request is currently being blocked by another session.

This query is useful for a quick check, but during real troubleshooting we usually need more information.

## Practical Blocking Troubleshooting Script

The following query combines SQL Server Dynamic Management Views to provide additional information about blocked requests.
```sql
SELECT
    r.session_id AS BlockedSessionID,
    r.blocking_session_id AS BlockingSessionID,

    DB_NAME(r.database_id) AS DatabaseName,

    s.login_name AS BlockedLogin,
    s.host_name AS BlockedHost,
    s.program_name AS BlockedProgram,

    r.status AS RequestStatus,
    r.command AS Command,

    r.wait_type AS WaitType,
    r.wait_time AS WaitTimeMS,
    r.wait_resource AS WaitResource,

    r.cpu_time AS CPUTimeMS,
    r.total_elapsed_time AS ElapsedTimeMS,

    r.reads,
    r.writes,
    r.logical_reads,

    SUBSTRING
    (
        st.text,
        (r.statement_start_offset / 2) + 1,

        (
            (
                CASE
                    WHEN r.statement_end_offset = -1
                        THEN DATALENGTH(st.text)
                    ELSE r.statement_end_offset
                END
                - r.statement_start_offset
            ) / 2
        ) + 1
    ) AS CurrentStatement,

    st.text AS FullBatchText

FROM sys.dm_exec_requests AS r

INNER JOIN sys.dm_exec_sessions AS s
    ON r.session_id = s.session_id

CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS st

WHERE
    r.blocking_session_id <> 0

ORDER BY
    r.wait_time DESC;
GO
```
### Understanding the Output

The most useful columns are:
```text
| Column              | Description                           |
| ------------------- | ------------------------------------- |
| `BlockedSessionID`  | Session currently waiting             |
| `BlockingSessionID` | Session causing the block             |
| `DatabaseName`      | Database associated with the request  |
| `BlockedLogin`      | Login used by the blocked session     |
| `BlockedHost`       | Client computer or application server |
| `BlockedProgram`    | Application connected to SQL Server   |
| `WaitType`          | Type of wait currently occurring      |
| `WaitTimeMS`        | Time the request has been waiting     |
| `WaitResource`      | Resource the request is waiting for   |
| `CurrentStatement`  | Statement currently executing         |
| `FullBatchText`     | Complete submitted SQL batch          |
```
For example:
```text
BlockedSessionID   BlockingSessionID   WaitType       WaitTimeMS
----------------   -----------------   ------------   ----------
72                 51                  LCK_M_X        18542
85                 72                  LCK_M_S        12677
```
This indicates the following blocking chain:
```text
51
 ↓
72
 ↓
85
```
Session `51` is therefore the likely head blocker.

### Find the Blocking Session Details

Once you identify the blocking session ID, investigate that session.
For example, if Session 51 is blocking other sessions:
```sql
SELECT
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    s.status,
    s.open_transaction_count,
    s.last_request_start_time,
    s.last_request_end_time
FROM sys.dm_exec_sessions AS s
WHERE s.session_id = 51;
GO
```
This helps determine whether the blocking session belongs to:

An application
A user running a manual query
A SQL Server Agent job
A session with an open transaction
A session that is sleeping but still holding locks

### Check the Last Command Executed

A blocking session may not always appear as an active request.
For example, a session can be in a sleeping state while still holding locks because an application opened a transaction but never committed or rolled it back.
To inspect the last command submitted by a session, use:
```sql
DBCC INPUTBUFFER(51);
GO
```
Replace `51` with the actual blocking session ID.

This can often reveal the SQL statement responsible for the open transaction.

### Check for Open Transactions

Long-running or abandoned transactions are common causes of blocking.
To check the oldest active transaction in the current database:
```sql
DBCC OPENTRAN;
GO
```
You can also look for sessions with open transactions:
```sql
SELECT
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    s.status,
    s.open_transaction_count
FROM sys.dm_exec_sessions AS s
WHERE s.open_transaction_count > 0
ORDER BY s.open_transaction_count DESC;
GO
```
Sessions with open transactions are not automatically a problem, but they deserve closer inspection if they are involved in prolonged blocking.

### Common Lock Wait Types

During blocking investigations, you may encounter wait types such as:
```text
LCK_M_S
LCK_M_X
LCK_M_U
LCK_M_IX
LCK_M_IS
```
A lock wait does not automatically mean SQL Server has a problem.
You should evaluate:
```text
Wait Duration
+
Frequency
+
Number of Blocked Sessions
+
Application Impact
```
### Example Blocking Scenario

Suppose Session 51 executes:
```sql
BEGIN TRANSACTION;

UPDATE dbo.Customer
SET CreditLimit = CreditLimit + 100
WHERE CustomerID = 1001;

-- Transaction intentionally left open
```
Session 51 now holds locks associated with this update.

A second session executes:
```sql
UPDATE dbo.Customer
SET CreditLimit = CreditLimit + 50
WHERE CustomerID = 1001;
```
The second session may now wait because Session 51 has not completed its transaction.

The blocking query might show:
```text
BlockedSessionID : 72
BlockingSessionID: 51
WaitType         : LCK_M_U
```
When Session 51 executes:
```sql
COMMIT TRANSACTION;
```
the lock can be released and the blocked session can continue.

### Should You Kill the Blocking Session?

SQL Server allows a DBA to terminate a session using:
```sql
KILL 51;
```
However, `KILL` should not be the first troubleshooting step.

Before terminating a blocking session, determine:

What application owns the session?
Is an important business transaction running?
How much work has already been performed?
How much rollback may be required?
Could terminating the session cause an application problem?
Is the blocking actually abnormal?

A large rollback can take significant time.

To check rollback progress:
```sql
KILL 51 WITH STATUSONLY;
GO
```
Use KILL only after understanding the potential impact.

## Why Blocking Happens

Common causes include:

### Long-Running Transactions

Transactions remain open for longer than necessary.

### Missing or Inefficient Indexes

Poor access paths can cause SQL Server to scan more data and hold locks longer.

### Large UPDATE or DELETE Operations

Large data modifications may hold locks for an extended period.

### Application Transaction Handling

Applications may begin a transaction and delay the commit or rollback.

### User Interaction Inside a Transaction

For example:
```sql
BEGIN TRANSACTION;

UPDATE dbo.SomeTable
SET SomeColumn = 'Value'
WHERE ID = 100;

-- User performs other work before COMMIT
```
Locks may remain held while the transaction is still open.

### Poor Query Performance

The longer a statement executes, the longer its locks may remain active.

## Blocking vs Deadlocking

Blocking and deadlocking are related, but they are not the same problem.

### Blocking
```text
Session A
   |
   v
Session B
```
Session B waits for Session A to release the required resource.

### Deadlock
```text
Session A
   ↓ waits for
Session B
   ↓ waits for
Session A
```
Neither session can continue.

SQL Server detects the deadlock and chooses one transaction as the deadlock victim.

Blocking does not automatically indicate that a deadlock exists.

## Required Permissions

Viewing server-wide Dynamic Management View information requires appropriate permissions.

Depending on the SQL Server version and environment, this commonly involves:
```sql
VIEW SERVER STATE
```
Newer SQL Server versions also use:
```sql
Newer SQL Server versions also use:
```
Always follow your organization's principle of least privilege.

## Production Troubleshooting Checklist

When investigating blocking, use a consistent process:

 Identify blocked session IDs
 Identify blocking session IDs
 Determine the head blocker
 Review wait type
 Review wait duration
 Check the wait resource
 Identify the database involved
 Review the SQL statement
 Review login, host, and application information
 Check for open transactions
 Determine whether blocking is temporary or persistent
 Review indexes and execution plans
 Check application transaction handling
 Avoid using KILL without understanding the impact
 Capture troubleshooting evidence before the blocking disappears

### Quick Reference Blocking Script

For quick troubleshooting, here is the primary query again:
```sql
SELECT
    r.session_id AS BlockedSessionID,
    r.blocking_session_id AS BlockingSessionID,

    DB_NAME(r.database_id) AS DatabaseName,

    s.login_name AS BlockedLogin,
    s.host_name AS BlockedHost,
    s.program_name AS BlockedProgram,

    r.status AS RequestStatus,
    r.command AS Command,

    r.wait_type AS WaitType,
    r.wait_time AS WaitTimeMS,
    r.wait_resource AS WaitResource,

    r.cpu_time AS CPUTimeMS,
    r.total_elapsed_time AS ElapsedTimeMS,

    r.reads,
    r.writes,
    r.logical_reads,

    SUBSTRING
    (
        st.text,
        (r.statement_start_offset / 2) + 1,

        (
            (
                CASE
                    WHEN r.statement_end_offset = -1
                        THEN DATALENGTH(st.text)
                    ELSE r.statement_end_offset
                END
                - r.statement_start_offset
            ) / 2
        ) + 1
    ) AS CurrentStatement,

    st.text AS FullBatchText

FROM sys.dm_exec_requests AS r

INNER JOIN sys.dm_exec_sessions AS s
    ON r.session_id = s.session_id

CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS st

WHERE
    r.blocking_session_id <> 0

ORDER BY
    r.wait_time DESC;
GO
```
## Conclusion

Blocking is a normal part of SQL Server concurrency, but prolonged blocking can create serious performance problems.

The goal of a DBA should not be simply to find a blocked session.

A better troubleshooting process is:
```text
Who is blocked?
      ↓
Who is blocking?
      ↓
What is the head blocker?
      ↓
What SQL is involved?
      ↓
Why are locks being held?
      ↓
What is the safest corrective action?
```
Using `sys.dm_exec_requests`, `sys.dm_exec_sessions`, and `sys.dm_exec_sql_text` provides a practical starting point for investigating SQL Server blocking directly with T-SQL.
