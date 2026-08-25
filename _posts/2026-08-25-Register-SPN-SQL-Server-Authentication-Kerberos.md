---
title: "Register a SPN for SQL Server Authentication with Kerberos"
description: "Ensuring your SQL Server connections authenticate with Kerberos rather than NTLM is a key part of hardening your database environment. Kerberos offers stronger, modern security guarantees and is the recommended authentication method for SQL Server deployments. This article outlines the required prerequisites and the steps needed to configure SQL Server to use Kerberos authentication.."
date: 2022-03-27
# last_modified_at: YYYY-MM-DD

categories:
  - SQL Server

tags:
  - SQL Server
  - Database Administration
  - Troubleshooting

image: /assets/images/social-preview.png
---

## Problem
After reviewing the `sys.dm_exec_connections` DMV, you may notice that all active sessions are using Windows Authentication with NTLM instead of Kerberos. The natural question is: How do I get SQL Server to authenticate using Kerberos?  
If you need background on SPN registration, this SPN overview is a good starting point.

## Solution Overview
Kerberos authentication in SQL Server depends on meeting a few foundational requirements. Once those prerequisites are in place, SQL Server can negotiate Kerberos instead of falling back to NTLM.

### Domain Requirements
Both the SQL Server host and the connecting clients must be joined to a domain. If they reside in different domains, those domains must have a two‑way trust established. This part is usually straightforward to verify.

### Service Principal Name (SPN)
SQL Server must have a valid SPN registered in Active Directory so clients can identify the service and request Kerberos tickets. SPN registration issues are the most common reason SQL Server silently falls back to NTLM.

### Checking Whether the SPN Is Registered
If SQL Server runs under a domain account (recommended), run:
```sql
setspn -l DOMAIN\SQLServiceAccount
```
If no SPNs are found, you’ll see an error similar to:
```sql
FindDomainForAccount: Call to DsGetDcNameWithAccountW failed with return value 0x00000525
Could not find account SQLServiceAccount
```

### SQL Server error log
Search the error log—either through SSMS filtering or `xp_read_errorlog`—for this message:

`The SQL Server Network Interface library could not register the Service Principal Name (SPN)… Failure to register an SPN may cause integrated authentication to fall back to NTLM instead of Kerberos.`

If this appears, SQL Server attempted automatic registration but failed.

### Active Directory inspection
Your AD administrator can also verify SPN entries directly using ADSIEdit.

## Fixing SPN Registration
Once you’ve identified the issue, you can choose one of two approaches to ensure SQL Server has the correct SPN.

### Option 1 — Automatic SPN Registration
SQL Server can register its SPN automatically at startup, but only if the service account has the necessary permissions:

 - Read servicePrincipalName
 - Write servicePrincipalName

These permissions must be granted in Active Directory.

Automatic registration also works when SQL Server runs under:
 - Local System
 - Network Service
 - A domain admin account
 - A domain account explicitly granted SPN write permissions

    ### Caution: Granting SPN write permissions is not recommended for clustered SQL Servers or environments with multiple domain controllers, because AD replication latency can cause intermittent connectivity issues.

### Option 2 — Manual SPN Registration
Manual registration uses the Microsoft setspn utility. You must be a domain admin or have delegated rights to manage SPNs.

Use the `-s` switch to ensure the SPN isn’t already defined.

#### Default instance
```sql
setspn -s MSSQLSvc/myhost.redmond.microsoft.com DOMAIN\SQLServiceAccount
```

#### Named instance
```sql
setspn -s MSSQLSvc/myhost.redmond.microsoft.com:instancename DOMAIN\SQLServiceAccount
```

## Verifying Kerberos Authentication
After registering the SPN and restarting SQL Server (if needed), establish a new connection and run:
```sql
select session_id, net_transport, client_net_address, auth_scheme
from sys.dm_exec_connections
```
If everything is configured correctly, `auth_scheme` will show **KERBEROS**.

## Frequently Asked Questions
### Kerberos vs NTLM — Why does SQL Server care?
NTLM is an older challenge‑response protocol that cannot delegate credentials. Kerberos uses ticket‑based authentication, supports delegation, and is required for scenarios like linked servers or multi‑tier applications. If Kerberos negotiation fails, SQL Server quietly falls back to NTLM.

### SPNs for Always On listeners — Does the listener need its own SPN?
Yes. The listener’s virtual network name is treated as a separate endpoint and requires its own SPN, registered under the same service account used by the replicas.

### Double hop problem — Does SPN registration fix it?
Not by itself. You also need to configure delegation on the SQL Server service account in Active Directory.

### Removing duplicate SPNs
Use:
 - `setspn -d` to delete a specific SPN
 - `setspn -x` to search for duplicates

Duplicate SPNs are a common cause of Kerberos failures.

### FQDN vs short name — Which should the SPN use?
It depends on how clients connect. If some use the short hostname and others use the FQDN, you need SPNs for both. Kerberos will not match a name that isn’t explicitly registered.

### Dynamic ports — Do they complicate SPNs?
Yes. Because SPNs include the port number, dynamic ports require re‑registering the SPN whenever SQL Server chooses a new port. Using a static port avoids this issue.
