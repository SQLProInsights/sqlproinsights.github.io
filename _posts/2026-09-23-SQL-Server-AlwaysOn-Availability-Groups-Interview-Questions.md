---
layout: post
title: "SQL Server AlwaysOn Availability Group Interview Questions"
date: 2024-05-23
categories:
  - SQL Server
tags:
  - SQL Server
  - Performance Tuning
  - DBA
  - Database
description: "SQL Server AlwaysOn Availability Group Interview Questions."
---

# SQL Server Always On Availability Groups Interview Preparation

If you are preparing for a SQL Server DBA interview where **Always On Availability Groups (AGs)** are heavily used, this guide provides a structured path from fundamentals to senior-level production troubleshooting.

The goal is not simply to memorize interview questions. You should be able to explain an AG architecture, describe how failover works, troubleshoot synchronization problems, discuss HA/DR design decisions, and reason through real production incidents.

**Target:** Be able to explain, troubleshoot, and design SQL Server Availability Groups in a production environment, including SQL Server 2025 considerations.

## Table of Contents

* [What You Should Be Able to Do](#what-you-should-be-able-to-do)
* [21-Day Preparation Roadmap](#21-day-preparation-roadmap)
* [Day 1 - Availability Group Architecture](#day-1---availability-group-architecture)
* [Day 2 - High Availability vs Disaster Recovery](#day-2---high-availability-vs-disaster-recovery)
* [Day 3 - Synchronous vs Asynchronous Commit](#day-3---synchronous-vs-asynchronous-commit)
* [Day 4 - Building an Availability Group](#day-4---building-an-availability-group)
* [Day 5 - Initial Synchronization and Seeding](#day-5---initial-synchronization-and-seeding)
* [Day 6 - Failover](#day-6---failover)
* [Day 7 - AG Listener and Connectivity](#day-7---ag-listener-and-connectivity)
* [Day 8 - Read-Only Routing](#day-8---read-only-routing)
* [Day 9 - Monitoring and DMVs](#day-9---monitoring-and-dmvs)
* [Day 10 - Synchronization Troubleshooting](#day-10---synchronization-troubleshooting)
* [Day 11 - Send Queue vs Redo Queue](#day-11---send-queue-vs-redo-queue)
* [Day 12 - AG Performance Troubleshooting](#day-12---ag-performance-troubleshooting)
* [Day 13 - Backup Strategy](#day-13---backup-strategy)
* [Day 14 - Transaction Log Growth and AGs](#day-14---transaction-log-growth-and-ags)
* [Day 15 - Patching and Upgrades](#day-15---patching-and-upgrades)
* [Day 16 - Distributed Availability Groups](#day-16---distributed-availability-groups)
* [Day 17 - SQL Server 2025 AG Topics](#day-17---sql-server-2025-ag-topics)
* [Day 18 - Advanced Troubleshooting Scenarios](#day-18---advanced-troubleshooting-scenarios)
* [Day 19 - Disaster Recovery Scenario](#day-19---disaster-recovery-scenario)
* [Day 20 - Mock Senior DBA Interview](#day-20---mock-senior-dba-interview)
* [Day 21 - Final Rapid-Fire Review](#day-21---final-rapid-fire-review)
* [AG SQL Scripts to Practice](#ag-sql-scripts-to-practice)
* [Must-Know DMVs](#must-know-dmvs)
* [Production Troubleshooting Framework](#production-troubleshooting-framework)
* [Final Interview Checklist](#final-interview-checklist)
* [Microsoft References](#microsoft-references)

## What You Should Be Able to Do

By the end of this preparation plan, you should be comfortable discussing these areas:


| Area | Interview expectation |
|---|---|
| AG Fundamentals | Explain architecture and terminology clearly |
| HA vs DR | Explain how AGs support availability and disaster recovery |
| Configuration | Describe how to build and configure an AG |
| Synchronization | Explain synchronous and asynchronous data movement |
| Failover | Explain automatic, planned, and forced failover |
| Monitoring | Use SSMS, DMVs, logs, and cluster information |
| Troubleshooting | Diagnose unhealthy or unsynchronized databases |
| Performance | Investigate send queues, redo queues, I/O, CPU, and network issues |
| DR | Explain RPO, RTO, site failure, and recovery procedures |
| SQL Server 2025 | Discuss relevant AG and Distributed AG changes |

The interview should be approached as a **production DBA problem-solving exercise**, not just a definition quiz.

---

# 21-Day Preparation Roadmap

| Day | Focus | Priority |
|---|---|---|
| 1 | AG architecture and terminology | High |
| 2 | HA vs DR, RPO/RTO | High |
| 3 | Synchronous vs asynchronous commit | High |
| 4 | AG installation and configuration | High |
| 5 | Seeding and synchronization | High |
| 6 | Failover | High |
| 7 | Listener and connectivity | High |
| 8 | Read-only routing | Medium |
| 9 | Monitoring and DMVs | High |
| 10 | Synchronization troubleshooting | High |
| 11 | Send queue vs redo queue | High |
| 12 | Performance troubleshooting | High |
| 13 | Backup strategy | Medium |
| 14 | Transaction log growth | High |
| 15 | Patching and upgrades | Medium |
| 16 | Distributed AG | Medium/High |
| 17 | SQL Server 2025 AG features | High |
| 18 | Advanced troubleshooting scenarios | High |
| 19 | DR scenario | High |
| 20 | Mock interview | High |
| 21 | Rapid-fire review | High |

---
# Day 1 - Availability Group Architecture

Always On Availability Groups are designed to provide high availability and disaster recovery for a defined set of databases.

A traditional architecture can be visualized as:
```text
                    Application
                         |
                         v
                  AG Listener
                         |
                +--------+--------+
                |                 |
                v                 v
          Primary Replica    Secondary Replica
             SQL01               SQL02
                |                 |
                +------ AG -------+
                       |
                  Data Movement
```

## Important terminology

Know these terms:

- Availability Group
- Availability Replica
- Primary Replica
- Secondary Replica
- Availability Database
- AG Listener
- Database Mirroring Endpoint
- Windows Server Failover Clustering (WSFC)
- Synchronous Commit
- Asynchronous Commit
- Automatic Failover
- Planned Manual Failover
- Forced Failover
- Readable Secondary
- Distributed Availability Group
- Contained Availability Group

On Windows, traditional Always On Availability Groups rely on WSFC to monitor and manage availability replica roles. Each availability group has a corresponding WSFC resource group. AGs themselves do not require shared storage, although an FCI used as an AG replica has its own shared-storage requirements. [Microsoft Learn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/failover-clustering-and-always-on-availability-groups-sql-server?view=sql-server-ver17)

## Interview questions

1. What is an Always On Availability Group?
2. What problem does an AG solve?
3. What is an availability replica?
4. What is the difference between a primary and secondary replica?
5. What is the purpose of the AG listener?
6. Why does a Windows-based AG use WSFC?
7. What is the database mirroring endpoint used for?
8. Does an AG require shared storage?
9. What is the difference between an AG and an FCI?

---

# Day 2 - High Availability vs Disaster Recovery

Understanding the difference between HA and DR is essential.

## High Availability

The objective is to reduce downtime when an individual server, SQL Server instance, or component fails.

Example:

```text
Production Site

SQL01
  |
  | Synchronous Commit
  v
SQL02
```

If SQL01 fails and the configured failover conditions are met, SQL02 can become the primary replica.

## Disaster Recovery

DR addresses larger failures such as:

- Data-center failure
- Major infrastructure outage
- Regional outage
- Loss of the production environment

Example:

```text
PRIMARY SITE                    DR SITE

SQL01                           SQL03
  |                                ^
  | Asynchronous Commit            |
  +--------------------------------+
```

## Know RPO and RTO

### RPO - Recovery Point Objective

How much data loss the business can tolerate.

### RTO - Recovery Time Objective

How quickly the service needs to be restored.

For example:

```text
RPO = 0
RTO = 60 seconds
```

This has architectural implications. You need to discuss synchronization mode, failover mode, network characteristics, secondary health, and application reconnection behavior.

## Interview questions

- What is the difference between HA and DR?
- What is RPO?
- What is RTO?
- How would you design for zero/minimal data loss?
- Why might asynchronous commit be appropriate for a DR replica?
- Can an AG protect against every type of disaster?

---

# Day 3 - Synchronous vs Asynchronous Commit

This is one of the most important AG interview topics.

## Synchronous Commit

Conceptually:

```text
Primary
   |
   | Log block
   v
Secondary
   |
   | Acknowledgment
   v
Primary transaction completes
```

The primary waits for the required acknowledgment from the synchronous secondary before the transaction can complete.

Advantages:

- Supports strong data-loss protection objectives
- Can support automatic failover when other requirements are met

Trade-off:

- Network latency and secondary performance can affect transaction latency

## Asynchronous Commit

```text
Primary
   |
   | Log blocks
   v
Secondary
```

The primary does not wait for the secondary to acknowledge the log block before completing the transaction.

Advantages:

- Better suited to geographically separated DR environments
- Less transaction latency impact from remote secondary latency

Trade-off:

- The secondary can lag behind the primary
- Forced DR failover can involve data loss

## Interview questions

> What is the difference between synchronous and asynchronous commit?

> Why would you use asynchronous commit?

> Does synchronous commit automatically mean automatic failover?

> Can an asynchronous secondary be used for DR?

> What happens to application transaction latency when synchronous secondary performance degrades?

---

# Day 4 - Building an Availability Group

Understand the complete implementation sequence.

```text
1. Prepare servers
       |
2. Configure WSFC
       |
3. Install SQL Server
       |
4. Enable Always On
       |
5. Configure database mirroring endpoints
       |
6. Create Availability Group
       |
7. Add replicas
       |
8. Prepare secondary databases
       |
9. Join secondary replicas
       |
10. Create AG listener
       |
11. Configure routing where required
       |
12. Test failover
```

Microsoft's getting-started guidance follows this general configuration process, including enabling Always On, ensuring the endpoint exists, creating the AG, configuring replicas, and managing databases and failover. [Microsoft Learn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/getting-started-with-always-on-availability-groups-sql-server?view=sql-server-ver17)

## Prerequisites to understand

- Supported SQL Server version/edition
- WSFC for Windows deployments
- Network connectivity
- SQL Server service accounts and permissions
- Database eligibility
- Database mirroring endpoint
- Firewall configuration
- DNS/listener requirements
- Disk/storage layout
- Backup/restore or seeding strategy

## Interview scenario

> "Walk me through building an AG for two SQL Server instances."

A strong answer should start with prerequisites rather than immediately jumping to `CREATE AVAILABILITY GROUP`.

---

# Day 5 - Initial Synchronization and Seeding

Before a secondary database can participate correctly, it must be initialized and joined.

Know these approaches:

- Automatic seeding
- Manual seeding
- Full backup/restore
- Transaction log restore
- `JOIN`
- `JOIN ONLY`

For a very large production database, consider:

- Backup duration
- Restore duration
- Network bandwidth
- Disk capacity
- Compression
- Transaction log generation during the initialization process
- Existing workload
- Maintenance window

## Interview scenario

> "You have a 4 TB production database and need to add a new secondary. How would you initialize it?"

A strong answer discusses the database size, backup/restore strategy, network capacity, disk layout, workload impact, seeding approach, and synchronization monitoring.

---

# Day 6 - Failover

Microsoft documents three forms of AG failover:

1. Automatic failover
2. Planned manual failover without data loss
3. Forced manual failover with possible data loss

[Microsoft Learn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/failover-and-failover-modes-always-on-availability-groups?view=sql-server-ver17)

## Automatic failover

Conceptually:

```text
Primary fails
     |
     v
WSFC / AG health detection
     |
     v
Eligible secondary becomes primary
```

Automatic failover depends on configured failover mode, synchronization state, cluster/AG health, and other requirements.

## Planned manual failover

Used for planned activities such as:

- Patching
- Maintenance
- Infrastructure work
- Controlled role changes

The target secondary should be appropriately synchronized.

## Forced failover

Used in emergency situations.

```text
Primary unavailable
       |
       v
Force secondary to primary
       |
       v
Potential data loss
```

## Interview questions

- What are the three types of failover?
- What are the prerequisites for automatic failover?
- When would you use planned manual failover?
- When would you use forced failover?
- Why can forced failover cause data loss?
- What do you do with the old primary after forced failover?
- How do you re-establish the AG after recovery?

---

# Day 7 - AG Listener and Connectivity

The application should normally connect through the AG listener rather than directly to a specific replica.

```text
Application
     |
     v
AG Listener
     |
     v
Current Primary
```

Instead of:

```text
Application --> SQL01
```

use:

```text
Application --> AGListener
```

## Learn

- Listener DNS name
- Listener IP
- Listener port
- Multi-subnet environments
- `MultiSubnetFailover=True`
- Client driver behavior
- DNS
- Firewall
- Connection retry behavior

## Interview scenario

> "The AG failed over successfully, but the application cannot connect. What do you check?"

A structured investigation:

1. Is the new primary healthy?
2. Is the listener online?
3. Does DNS resolve correctly?
4. Can the client reach the listener IP?
5. Is the SQL port accessible?
6. Is the application using the correct connection string?
7. Is `MultiSubnetFailover=True` appropriate for the deployment?
8. Are there application connection retry issues?

---

# Day 8 - Read-Only Routing

Readable secondary replicas can be used to offload appropriate read workloads.

Conceptually:

```text
                    AG Listener
                        |
          +-------------+-------------+
          |                           |
       Read/Write                  Read Only
          |                           |
          v                           v
       Primary                  Readable Secondary
```

Important concepts:

- `ApplicationIntent=ReadOnly`
- Read-only routing URL
- Routing list
- Listener
- Secondary read access
- Application connection string
- Secondary workload monitoring

## Interview question

> "How would you offload reporting queries from the primary?"

Discuss:

1. Readable secondary
2. Read-only access
3. Read-only routing
4. Application connection settings
5. Monitoring the secondary's workload

---

# Day 9 - Monitoring and DMVs

A production DBA should be able to monitor AGs without relying only on the SSMS dashboard.

Important catalog views and DMVs include:

```sql
sys.availability_groups

sys.availability_replicas

sys.availability_databases_cluster

sys.dm_hadr_availability_replica_states

sys.dm_hadr_database_replica_states

sys.dm_hadr_database_replica_cluster_states

sys.dm_hadr_availability_replica_cluster_states
```

Useful concepts include:

```text
role_desc
connected_state
operational_state
synchronization_state
synchronization_health
database_state
suspend_reason
log_send_queue_size
redo_queue_size
redo_rate
```

Also know how to use:

- SSMS Always On Dashboard
- SQL Server error log
- Windows Event Viewer
- WSFC tools
- Extended Events
- Performance Monitor
- SQL Agent monitoring
- Enterprise monitoring platforms

---

# Day 10 - Synchronization Troubleshooting

Know the major synchronization states:

```text
SYNCHRONIZED
SYNCHRONIZING
NOT SYNCHRONIZING
REVERTING
INITIALIZING
```

When a database is not synchronizing, don't immediately restart SQL Server.

Use a structured approach.

## Troubleshooting flow

```text
Database not synchronized
          |
          v
Check database state
          |
          v
Check replica connection state
          |
          v
Check synchronization state
          |
          v
Check suspend reason
          |
          v
Check SQL error log
          |
          v
Check endpoint
          |
          v
Check network
          |
          v
Check storage / CPU / workload
```

Microsoft's troubleshooting guidance specifically covers common issues involving AG enablement, accounts, endpoints, network access, listeners, endpoint access, database joins, and read-only routing. [Microsoft Learn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/troubleshoot-always-on-availability-groups-configuration-sql-server?view=sql-server-ver17)

---

# Day 11 - Send Queue vs Redo Queue

This is an important senior-level concept.

Think of the data movement path as:

```text
PRIMARY
   |
   | Generate log
   v
SEND
   |
   v
SECONDARY
   |
   | REDO
   v
SECONDARY DATA FILES
```

## Send Queue

Log records that have not yet been sent to the secondary.

Potential causes:

- Network problems
- Endpoint problems
- Secondary connectivity
- High log generation rate
- Secondary unable to receive data fast enough

## Redo Queue

Log records received by the secondary but not yet applied to the secondary database.

Potential causes:

- Secondary CPU pressure
- Secondary storage latency
- Redo bottleneck
- Large transactions
- Secondary workload

## Interview scenario

> "The secondary has a large redo queue. What do you investigate?"

Discuss:

- Secondary CPU
- Disk latency
- I/O throughput
- Redo rate
- Transaction size
- Workload running on secondary
- Database state
- Blocking or resource contention where relevant

---

# Day 12 - AG Performance Troubleshooting

## Scenario 1 - Send queue is growing

```text
Primary
   |
   | High send queue
   v
Secondary
```

Investigate:

- Network bandwidth
- Network latency
- Endpoint connectivity
- Primary log generation rate
- Secondary connection state
- Secondary receive capability

## Scenario 2 - Redo queue is growing

Investigate:

- Secondary CPU
- Storage latency
- I/O throughput
- Redo rate
- Secondary workload
- Large transactions

## Scenario 3 - Synchronous commit is causing transaction latency

Investigate:

- Network latency
- Secondary disk latency
- Secondary CPU
- Commit latency
- Synchronization health
- Recent workload changes

Do not immediately switch synchronous to asynchronous without considering the business RPO and HA/DR design.

---

# Day 13 - Backup Strategy

An Availability Group does **not** replace a backup strategy.

Understand:

- Full backups
- Differential backups
- Transaction log backups
- Copy-only backups
- Backup preference
- Preferred backup replica
- Backup history
- Restore testing

## Interview question

> "Can backups be taken from a secondary replica?"

Your answer should discuss supported backup types, configured backup preference, workload considerations, and how the backup strategy fits into the overall recovery plan.

A strong DBA answer also makes the distinction:

```text
AG replication != Backup
```

An AG helps maintain availability and copies database changes between replicas. Backups are still required for recovery scenarios such as accidental deletion, corruption, point-in-time recovery, and other operational needs.

---

# Day 14 - Transaction Log Growth and AGs

A common production scenario is:

> "The transaction log on the primary is growing rapidly. The database participates in an AG. What do you check?"

Do not assume the AG is automatically the cause.

Investigate:

1. `log_reuse_wait_desc`
2. Transaction log backup health
3. Secondary synchronization state
4. Send queue
5. Redo queue
6. Suspended data movement
7. Secondary availability
8. Long-running transactions
9. Transaction generation rate
10. Disk capacity

Conceptually:

```text
Primary Transaction Log
          |
          +---- Log Backup
          |
          +---- AG Data Movement
                         |
                         v
                    Secondary
```

---

# Day 15 - Patching and Upgrades

Always On AGs are particularly useful for planned maintenance when the environment is designed and tested correctly.

A simplified rolling maintenance concept:

```text
Primary
   |
Secondary
   |
Patch secondary
   |
Fail over
   |
Secondary becomes primary
   |
Patch old primary
```

Understand:

- SQL Server version compatibility
- Cumulative Updates
- Security updates
- Major-version upgrades
- Failover direction
- Synchronization requirements
- Application validation
- Rollback planning

## Interview scenario

> "How would you patch a two-node AG with minimal application downtime?"

Your answer should explain the sequence, health checks, synchronization validation, planned failover, patching of the former primary, and post-maintenance validation.

---

# Day 16 - Distributed Availability Groups

A Distributed Availability Group spans two separate Availability Groups.

```text
                 Distributed AG
                       |
             +---------+---------+
             |                   |
             v                   v
            AG1                 AG2
       Primary Site          DR Site
```

The underlying AGs can be in different locations and can be on separate clusters. The distributed AG itself is maintained by SQL Server rather than being represented as one WSFC resource spanning both clusters. [Microsoft Learn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/distributed-availability-groups?view=sql-server-ver17)

## Know these terms

- Global primary
- Forwarder
- Underlying availability group
- Distributed AG
- Cross-site synchronization
- DR
- Manual failover

Important interview point:

> A distributed AG is not simply a larger four-node traditional AG.

It connects two separate availability groups.

---

# Day 17 - SQL Server 2025 AG Topics

If the job specifically mentions SQL Server 2025, spend dedicated preparation time here.

## 1. Distributed AG synchronization improvement

SQL Server 2025 introduces a change to the internal synchronization mechanism for distributed AGs intended to improve synchronization performance by reducing network saturation when the forwarder replica uses asynchronous commit. Microsoft documents this behavior as enabled by default and requiring no configuration. [Microsoft Learn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/distributed-availability-groups?view=sql-server-ver17)

### Interview question

> "What SQL Server 2025 improvements to Distributed Availability Groups are you aware of?"

---

## 2. Distributed contained Availability Groups

SQL Server 2025 adds support for distributed availability groups involving contained availability groups. Microsoft documents the requirement to use `AUTOSEEDING_SYSTEM_DATABASES` when a contained AG is used as the forwarder. [Microsoft Learn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/distributed-availability-groups?view=sql-server-ver17)

---

## 3. Contained Availability Groups

Contained AGs support AG-level management of metadata such as users, logins, permissions, and SQL Agent jobs, along with specialized contained system databases.

Contained AGs are available in SQL Server 2022 and later, and SQL Server 2025 adds distributed contained AG support. [Microsoft Learn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/contained-availability-groups-overview?view=sql-server-ver17)

Conceptually:

```text
Contained AG
   |
   +-- User databases
   |
   +-- Contained master
   |
   +-- Contained msdb
```

This is important because configuration metadata can travel with the contained AG.

---

## 4. Fast Failover

SQL Server 2025 also introduces changes around AG failover behavior. Review the current Microsoft documentation for `RestartThreshold` and the SQL Server 2025 failover behavior before the interview.

### Interview question

> "What changed in SQL Server 2025 around AG failover behavior?"

Be prepared to explain the health-detection/failover sequence and the operational implications rather than just naming the feature.

[Microsoft Learn - Failover and Failover Modes](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/failover-and-failover-modes-always-on-availability-groups?view=sql-server-ver17)

---

# Day 18 - Advanced Troubleshooting Scenarios

Practice answering these without looking at notes.

## Scenario 1

> The secondary database shows `NOT SYNCHRONIZING`.

Explain exactly what you would check.

---

## Scenario 2

> The transaction log on the primary is growing rapidly.

Explain how you would determine whether the AG is contributing to the problem.

---

## Scenario 3

> The AG listener is unavailable.

Walk through the troubleshooting layers:

```text
DNS
  |
Listener
  |
IP
  |
Port
  |
Firewall
  |
SQL Server
  |
AG
  |
WSFC
```

---

## Scenario 4

> Automatic failover did not occur.

Check:

```text
WSFC health
Replica configuration
Synchronization state
Failover mode
Cluster health
SQL Server health
Connectivity
Failover policy
```

---

## Scenario 5

> The secondary is synchronized but reporting queries are slow.

Investigate:

- Secondary CPU
- Secondary I/O
- Query workload
- Read-only routing
- Connection routing
- Statistics/query behavior
- Resource contention

---

## Scenario 6

> Network latency between primary and secondary suddenly increases.

Discuss:

- Synchronous commit latency
- Send queue
- Commit latency
- RPO implications
- Network troubleshooting
- Whether an architectural change is appropriate

Do not make an immediate configuration change without understanding the business requirements.

---

# Day 19 - Disaster Recovery Scenario

Imagine this interview question:

> **"The primary data center is completely unavailable. You have an asynchronous secondary in another region. Walk me through your DR procedure."**

A structured answer:

```text
1. Confirm primary-site failure
2. Confirm DR replica health
3. Determine synchronization state
4. Determine potential data loss
5. Understand business RPO
6. Obtain required operational/business approval
7. Perform the appropriate failover procedure
8. Redirect/validate application connectivity
9. Validate databases
10. Validate application functionality
11. Monitor the new primary
12. Plan recovery of the original primary environment
13. Re-establish the HA/DR topology
```

The important part is that you don't simply say:

> "I would fail over."

Explain **why**, **what evidence you would check**, **what risk exists**, and **how you would validate the result**.

---

# Day 20 - Mock Senior DBA Interview

Practice answering these aloud.

## Fundamentals

1. What is Always On Availability Group?
2. AG vs FCI?
3. AG vs database mirroring?
4. What is WSFC?
5. What is the AG listener?

## Architecture

6. Explain synchronous commit.
7. Explain asynchronous commit.
8. How does transaction log data move from primary to secondary?
9. What happens during failover?
10. What happens when the secondary becomes unavailable?

## Configuration

11. How do you create an AG?
12. What are the prerequisites?
13. How do you initialize a secondary?
14. How do you configure the listener?
15. How do you configure read-only routing?

## Troubleshooting

16. Why is a database not synchronizing?
17. What is the send queue?
18. What is the redo queue?
19. Why would a transaction log grow?
20. How do you troubleshoot synchronization latency?

## HA/DR

21. Automatic vs manual failover?
22. Planned vs forced failover?
23. How do you perform DR?
24. How do you minimize data loss?
25. How would you design AG for two data centers?

## SQL Server 2025

26. What AG changes are you aware of in SQL Server 2025?
27. What is fast failover?
28. What changed with Distributed AG synchronization?
29. What is a contained AG?
30. What is a distributed contained AG?

---

# Day 21 - Final Rapid-Fire Review

Before the interview, make sure you can answer these immediately:

### Architecture

- What is an AG?
- What is a replica?
- What is a listener?
- What is WSFC?
- Does AG require shared storage?

### Synchronization

- Synchronous vs asynchronous?
- Send queue?
- Redo queue?
- Synchronization state?
- Synchronization health?

### Failover

- Automatic?
- Planned manual?
- Forced?
- Data-loss risk?
- Failover prerequisites?

### Troubleshooting

- Database not synchronizing?
- Listener unavailable?
- Endpoint error?
- Network latency?
- Redo queue growing?
- Transaction log growing?

### HA/DR

- RPO?
- RTO?
- Same-site HA?
- Cross-site DR?
- Synchronous DR?
- Asynchronous DR?

### SQL Server 2025

- Fast failover?
- Distributed AG improvements?
- Contained AG?
- Distributed contained AG?

---

# AG SQL Scripts to Practice

Since an AG-focused DBA role is the target, maintain a dedicated collection of scripts.

A useful repository structure is:

```text
07-AlwaysOn/
│
├── 01-AG-Health-Checks.sql
├── 02-AG-Replica-Status.sql
├── 03-AG-Database-Synchronization.sql
├── 04-AG-Send-Queue.sql
├── 05-AG-Redo-Queue.sql
├── 06-AG-Listener.sql
├── 07-AG-Failover-Readiness.sql
├── 08-AG-Database-States.sql
├── 09-AG-Log-Growth-Troubleshooting.sql
├── 10-AG-Performance-Troubleshooting.sql
├── 11-AG-Backup-Status.sql
├── 12-AG-Failover.sql
├── 13-AG-Read-Only-Routing.sql
├── 14-AG-Distributed-AG.sql
└── 15-AG-2025-Features.sql
```

This gives you both an interview study resource and a reusable DBA toolkit.

---

# Must-Know DMVs

Be comfortable recognizing these immediately:

```sql
sys.availability_groups

sys.availability_replicas

sys.availability_databases_cluster

sys.dm_hadr_availability_replica_states

sys.dm_hadr_database_replica_states

sys.dm_hadr_database_replica_cluster_states

sys.dm_hadr_availability_replica_cluster_states
```

Also know the purpose of these commonly examined columns/concepts:

```text
role_desc
connected_state
operational_state
synchronization_state
synchronization_health
database_state
suspend_reason
log_send_queue_size
redo_queue_size
redo_rate
```

The exact DMV query should depend on the troubleshooting question. Avoid memorizing one giant query without understanding what each value means.

---

# Production Troubleshooting Framework

For almost every AG troubleshooting question, use the same structured method.

## 1. Establish the symptom

> What exactly is failing?

Examples:

- Application cannot connect
- Secondary not synchronized
- Failover failed
- Transaction log growing
- Queries are slow
- DR replica is behind

## 2. Check AG health

Look at:

- Replica role
- Connection state
- Synchronization state
- Synchronization health
- Database state
- Suspend reason

## 3. Identify the layer

Use this mental model:

```text
Application
     |
     v
Listener
     |
     v
SQL Server
     |
     v
Availability Group
     |
     v
Endpoint
     |
     v
Network
     |
     v
WSFC
     |
     v
Storage
```

## 4. Gather evidence

Use:

- SSMS
- AG Dashboard
- DMVs
- SQL Server error log
- Windows Event Viewer
- WSFC tools
- Extended Events
- Performance Monitor

## 5. Determine business impact

Ask:

- Is production available?
- Is there data-loss risk?
- Is RPO affected?
- Is RTO affected?
- Is the primary healthy?
- Is the secondary healthy?

## 6. Correct the root cause

Avoid making configuration changes before understanding the failure.

## 7. Validate

After remediation, verify:

```text
AG health
Database synchronization
Application connectivity
Performance
Backups
Monitoring
```

---

# AG Interview Answer Framework

A strong senior DBA answer generally follows this pattern:

> **Identify → Check → Isolate → Correct → Validate**

For example:

> "If a secondary is not synchronizing, I would first identify whether the issue is database-level, replica-level, endpoint/network-related, or resource-related. I would check the synchronization state, synchronization health, connection state, suspend reason, send and redo queues, SQL Server error log, endpoint connectivity, network health, and secondary CPU/I/O. Once the root cause is identified, I would correct it and then validate synchronization, database health, application impact, and monitoring."

This is much stronger than simply saying:

> "I would restart the secondary."

---

# Interview Preparation: What to Prioritize

## Tier 1 - Must Know

1. AG architecture
2. Synchronous vs asynchronous commit
3. Automatic vs planned vs forced failover
4. AG listener
5. WSFC
6. Synchronization states
7. Send queue vs redo queue
8. AG troubleshooting
9. RPO/RTO
10. DR scenarios

## Tier 2 - Senior DBA

11. Read-only routing
12. Backup strategy
13. Transaction log growth
14. Rolling patching
15. Distributed AG
16. Performance troubleshooting

## Tier 3 - SQL Server 2025

17. SQL Server 2025 failover changes
18. Distributed AG synchronization improvements
19. Contained AG
20. Distributed contained AG

---

# Final Interview Checklist

Before the interview, you should be able to explain the following **without referring to notes**:

- [ ] Draw a two-replica AG architecture
- [ ] Explain the role of WSFC
- [ ] Explain the AG listener
- [ ] Explain synchronous commit
- [ ] Explain asynchronous commit
- [ ] Explain automatic failover
- [ ] Explain planned manual failover
- [ ] Explain forced failover
- [ ] Explain possible data loss
- [ ] Explain RPO and RTO
- [ ] Explain send queue
- [ ] Explain redo queue
- [ ] Troubleshoot `NOT SYNCHRONIZING`
- [ ] Troubleshoot an unavailable listener
- [ ] Troubleshoot an endpoint problem
- [ ] Explain read-only routing
- [ ] Explain AG backup strategy
- [ ] Explain transaction log growth in an AG
- [ ] Explain rolling patching
- [ ] Explain Distributed AG
- [ ] Explain contained AG
- [ ] Explain SQL Server 2025 AG changes
- [ ] Walk through a complete DR scenario
- [ ] Write basic AG DMV queries
- [ ] Explain your troubleshooting methodology

---

# Microsoft References

The following Microsoft Learn resources should be your primary technical references while preparing:

- [Getting Started with Always On Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/getting-started-with-always-on-availability-groups-sql-server?view=sql-server-ver17)
- [Failover and Failover Modes](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/failover-and-failover-modes-always-on-availability-groups?view=sql-server-ver17)
- [Distributed Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/distributed-availability-groups?view=sql-server-ver17)
- [Contained Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/contained-availability-groups-overview?view=sql-server-ver17)
- [Availability Group Troubleshooting](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/troubleshoot-always-on-availability-groups-configuration-sql-server?view=sql-server-ver17)
- [Failover Clustering and Always On Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/failover-clustering-and-always-on-availability-groups-sql-server?view=sql-server-ver17)
- [CREATE AVAILABILITY GROUP](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-availability-group-transact-sql?view=sql-server-ver17)
- [SQL Server 2025 Editions and Supported Features](https://learn.microsoft.com/en-us/sql/sql-server/editions-and-components-of-sql-server-2025?view=sql-server-ver17)

---

## Final Goal

The objective of this preparation is not to memorize 30 answers.

It is to reach the point where an interviewer can give you a production incident such as:

> **"The primary is healthy, the secondary is connected but the redo queue has been growing for 30 minutes, the transaction log is growing, and the application team reports increased latency. What do you do?"**

…and you can calmly walk through:

```text
Symptom
   ↓
AG Health
   ↓
Synchronization
   ↓
Send / Redo Queues
   ↓
CPU / Memory / I/O
   ↓
Network
   ↓
Transaction Log
   ↓
Root Cause
   ↓
Corrective Action
   ↓
Validation
```


**SQL Pro Insights**
*Practical Technology. Real-World Solutions.*
