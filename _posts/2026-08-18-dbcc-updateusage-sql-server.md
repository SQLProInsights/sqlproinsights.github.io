---
layout: post
title: "DBCC UPDATEUSAGE in SQL Server: What It Does, When to Run It, and When Not To"
date: 2026-08-18
categories:
  - SQL Server
tags:
  - SQL Server
  - DBCC
  - Database Administration
  - Troubleshooting
  - Migration
description: "Learn what DBCC UPDATEUSAGE does in SQL Server, when you should run it, when you should not, and whether it is necessary after a database migration or restore."
---

`DBCC UPDATEUSAGE` is one of those SQL Server commands that many DBAs encounter during migrations, upgrades, or troubleshooting but may rarely need during normal database administration.

A common question is:

> **Should I run DBCC UPDATEUSAGE after restoring or migrating a database to a new SQL Server?**

Usually, the answer is **no — not automatically**.

Let's understand why.

## 1. What Does DBCC UPDATEUSAGE Do?

`DBCC UPDATEUSAGE` checks and corrects inaccuracies in SQL Server's metadata related to page and row counts.

These counts are used when SQL Server reports information such as:

- Number of rows
- Reserved pages
- Used pages
- Data pages
- Space utilization

Incorrect metadata can cause commands such as `sp_spaceused` to report inaccurate information.

A basic execution against the current database is:

```sql
DBCC UPDATEUSAGE (0);
```

Here, `0` means the **current database**.

## 2. Basic Syntax

The general syntax is:

```sql
DBCC UPDATEUSAGE
(
    { database_name | database_id | 0 }
    [ , { table_name | table_id | view_name | view_id }
    [ , { index_name | index_id } ] ]
)
[ WITH NO_INFOMSGS ];
```

You can therefore run the command against:

- An entire database
- A specific table
- An indexed view
- A specific index

## 3. Running It Against an Entire Database

For example:

```sql
USE YourDatabase;
GO

DBCC UPDATEUSAGE (0);
GO
```

To suppress informational messages:

```sql
DBCC UPDATEUSAGE (0)
WITH NO_INFOMSGS;
GO
```

For a large production database, remember that processing the entire database can take time.

## 4. Running It Against a Specific Table

If you suspect the problem is limited to one table, you don't necessarily need to process the entire database.

For example:

```sql
DBCC UPDATEUSAGE
(
    YourDatabase,
    'dbo.YourTable'
);
GO
```

This can be preferable when troubleshooting a specific object.

## 5. Why Would Page or Row Counts Become Incorrect?

SQL Server normally maintains this metadata automatically.

However, incorrect counts can sometimes exist, particularly with older databases or unusual metadata conditions.

Symptoms can include:

- Incorrect row counts
- Incorrect page counts
- Incorrect space usage information
- Unexpected `sp_spaceused` results
- A recommendation from `DBCC CHECKDB` to run `DBCC UPDATEUSAGE`

The important point is that `DBCC UPDATEUSAGE` is primarily a **metadata correction command**.

It is not a general-purpose performance tuning command.

## 6. How Does DBCC CHECKDB Relate to UPDATEUSAGE?

`DBCC CHECKDB` performs integrity checks against a database.

For example:

```sql
DBCC CHECKDB ('YourDatabase')
WITH NO_INFOMSGS;
GO
```

If SQL Server detects certain incorrect page or row counts, `DBCC CHECKDB` can report the issue and recommend running `DBCC UPDATEUSAGE`.

In that situation, running `DBCC UPDATEUSAGE` is appropriate.

For example:

```sql
USE YourDatabase;
GO

DBCC UPDATEUSAGE (0)
WITH NO_INFOMSGS;
GO
```

You can then run `DBCC CHECKDB` again:

```sql
DBCC CHECKDB ('YourDatabase')
WITH NO_INFOMSGS;
GO
```

## 7. Should DBCC UPDATEUSAGE Be Part of Regular Maintenance?

Generally, **no**.

SQL Server normally maintains page and row count metadata itself.

Therefore, running this command every day or after every maintenance operation is usually unnecessary.

A better approach is to run it when:

- `DBCC CHECKDB` recommends it
- `sp_spaceused` appears inaccurate
- You have evidence that page or row count metadata is incorrect
- You are troubleshooting a specific metadata discrepancy

Avoid treating it as a routine "just in case" maintenance command.

## 8. Should You Run It After Restoring a Database?

This is an important DBA question.

Suppose you perform:

```text
SQL Server
    ↓
Full Backup
    ↓
Restore to another SQL Server
```

Do you automatically need:

```sql
DBCC UPDATEUSAGE (0);
```

**No.**

A normal backup and restore does not by itself mean that the database's usage metadata is incorrect.

Instead, perform normal post-restore validation.

For example:

```sql
DBCC CHECKDB ('YourDatabase')
WITH NO_INFOMSGS;
GO
```

If `DBCC CHECKDB` reports no relevant metadata problem, there is normally no reason to run `DBCC UPDATEUSAGE` simply because the database was restored.

## 9. What About a SQL Server Version Upgrade?

Consider a migration such as:

```text
SQL Server 2016
        ↓
SQL Server 2025
```

Again, I would **not automatically run DBCC UPDATEUSAGE solely because the database moved to a newer SQL Server version**.

Instead, validate the database systematically.

A post-migration process should include:

1. Confirm the database is online.
2. Run appropriate integrity checks.
3. Validate application connectivity.
4. Validate SQL Server Agent jobs.
5. Validate logins and permissions.
6. Review application performance.
7. Check for SQL Server errors or warnings.
8. Investigate any metadata discrepancies.

Run `DBCC UPDATEUSAGE` if the validation process indicates that it is needed.

## 10. DBCC UPDATEUSAGE Is Not UPDATE STATISTICS

This distinction is very important.

These commands solve different problems.

### DBCC UPDATEUSAGE

Corrects metadata related to things such as:

- Row counts
- Used pages
- Reserved pages
- Data pages

Example:

```sql
DBCC UPDATEUSAGE (0);
```

### UPDATE STATISTICS

Updates optimizer statistics used for query optimization.

Example:

```sql
UPDATE STATISTICS dbo.YourTable;
```

Or:

```sql
EXEC sys.sp_updatestats;
```

Therefore:

```text
DBCC UPDATEUSAGE
        ≠
UPDATE STATISTICS
```

Running `DBCC UPDATEUSAGE` should not be considered a replacement for statistics maintenance.

## 11. DBCC UPDATEUSAGE Is Not DBCC CHECKDB

These commands also serve different purposes.

### DBCC CHECKDB

Checks the logical and physical integrity of database objects.

```sql
DBCC CHECKDB ('YourDatabase');
```

### DBCC UPDATEUSAGE

Corrects inaccurate page and row count metadata.

```sql
DBCC UPDATEUSAGE (0);
```

A useful way to remember the difference is:

```text
CHECKDB
   ↓
Database integrity

UPDATEUSAGE
   ↓
Usage metadata
```

## 12. Using sp_spaceused

If you're investigating space usage, you might start with:

```sql
EXEC sys.sp_spaceused;
GO
```

For a particular table:

```sql
EXEC sys.sp_spaceused
    @objname = N'dbo.YourTable';
GO
```

If the reported values appear incorrect, usage metadata may need investigation.

`sp_spaceused` also provides an option that can update usage information:

```sql
EXEC sys.sp_spaceused
    @updateusage = N'true';
GO
```

However, this should also be used deliberately rather than automatically on production databases.

## 13. What Does COUNT_ROWS Do?

`DBCC UPDATEUSAGE` supports the `COUNT_ROWS` option.

For example:

```sql
DBCC UPDATEUSAGE (0)
WITH COUNT_ROWS;
GO
```

This causes SQL Server to update the row-count information using the current number of rows.

Be careful with this option on large databases because obtaining row counts can increase the amount of work required.

## 14. Production Considerations

Before running `DBCC UPDATEUSAGE` across a large production database, consider:

- Database size
- Number of tables and indexes
- Current workload
- Maintenance window
- Whether the issue affects one object or the entire database
- Whether you actually have evidence of incorrect metadata

If only one table has a problem, target that object rather than automatically processing the entire database.

## 15. A Practical DBA Decision Process

A simple decision process can look like this:

```text
Are space/page/row counts suspicious?
                |
           No --+--> Don't run UPDATEUSAGE
                |
               Yes
                |
                v
       Validate the problem
                |
                v
     Run DBCC CHECKDB if appropriate
                |
                v
Does CHECKDB recommend UPDATEUSAGE
or is metadata clearly incorrect?
                |
           No --+--> Investigate further
                |
               Yes
                |
                v
       Run DBCC UPDATEUSAGE
                |
                v
          Validate again
```

The key principle is:

> **Use DBCC UPDATEUSAGE because you have a reason to use it, not simply because a database was restored or migrated.**

## 16. Recommended Post-Migration Approach

For a migration from SQL Server 2016 to SQL Server 2025, a reasonable high-level validation sequence is:

```sql
-- Confirm database status

SELECT
    name,
    state_desc,
    compatibility_level
FROM sys.databases
WHERE name = 'YourDatabase';
GO
```

Then perform an appropriate integrity check:

```sql
DBCC CHECKDB ('YourDatabase')
WITH NO_INFOMSGS;
GO
```

Validate the application and monitor the environment.

If `DBCC CHECKDB` reports incorrect page or row counts and recommends `DBCC UPDATEUSAGE`, then run:

```sql
USE YourDatabase;
GO

DBCC UPDATEUSAGE (0)
WITH NO_INFOMSGS;
GO
```

Then validate again.

## Best Practices

- Don't run `DBCC UPDATEUSAGE` routinely without a reason.
- Don't confuse it with `UPDATE STATISTICS`.
- Don't use it as a substitute for `DBCC CHECKDB`.
- Use it when metadata counts are known or suspected to be inaccurate.
- Pay attention when `DBCC CHECKDB` explicitly recommends it.
- Consider targeting an individual table instead of the whole database.
- Be cautious when running it against very large production databases.
- Validate the database again after correcting a reported problem.

## Related Articles

If you're planning a SQL Server upgrade or migration, see:

- [SQL Server 2016 to SQL Server 2025 Migration: Complete Step-by-Step Guide](/blog/sql-server-2016-to-2025-migration/)

- ## Final Thoughts

`DBCC UPDATEUSAGE` is a useful SQL Server administration command, but it is **not something that needs to run after every backup, restore, migration, or upgrade**.

SQL Server normally maintains usage metadata automatically.

The command becomes useful when page or row counts are incorrect, space reporting is suspicious, or SQL Server specifically identifies a usage metadata problem.

For a SQL Server 2016 to SQL Server 2025 migration, focus first on **integrity checks, application validation, security, dependencies, and performance testing**.

Run `DBCC UPDATEUSAGE` when the evidence tells you it is necessary — not simply because the database moved to a new server.
