<!--
Métadonnées du document (invisibles dans le rendu)
title: Guide: Migrate SQL Server Azure IASS 2019 to 2022
description: migrate SQL Server Azure IASS 2019 to 2022 
keywords: azure
category: azure
subcategory: 
document_type: guide
language: en
language_name: English
version: 1.0
status: production
generated_date: 2025-12-15T17:02:26.287Z
data_source: 
data_composition: official_only
enriched: 
sources_count: 10
direct_links_used: 
verification_status: verified
-->

# Guide: Migrate SQL Server Azure IASS 2019 to 2022

![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Lang](https://img.shields.io/badge/Lang-EN-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production-green?style=for-the-badge)
![Data](https://img.shields.io/badge/Data-100%25-success?style=for-the-badge)

---

## 📋 Métadonnées du document
🔗 LIENS CRITIQUES (à mentionner dans la doc):
- Azure DMS: SQL Server to Azure SQL Database: https://learn.microsoft.com/en-us/azure/dms/tutorial-sql-server-to-azure-sql (catégorie: sql-server-azure)
- Azure DMS: SQL Server to Managed Instance (Online): https://learn.microsoft.com/en-us/azure/dms/tutorial-sql-server-managed-instance-online (catégorie: sql-server-azure)
- 🚨 CRITICAL: SQL Server Login Management: https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/create-a-login (catégorie: sql-logins-critical)
- Transfer Logins and Passwords Between SQL Server Instances: https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/security/transfer-logins-passwords-between-instances (catégorie: sql-server-logins)
- 🚨 CRITICAL: Troubleshoot Orphaned Users: https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/troubleshoot-orphaned-users-sql-server (catégorie: sql-orphaned-users)
- 🚨 CRITICAL: SQL Server Agent Jobs: https://learn.microsoft.com/en-us/sql/ssms/agent/sql-server-agent (catégorie: sql-jobs-critical)
- 🚨 CRITICAL: Linked Servers Configuration: https://learn.microsoft.com/en-us/sql/relational-databases/linked-servers/linked-servers-database-engine (catégorie: sql-linked-servers)

---

## ⚠️ IMPORTANT NOTICE

**⚠️ INFORMATION NON DISPONIBLE**

No official, detailed, step-by-step migration procedure, scripts, commands, or configuration guidance specifically for migrating SQL Server 2019 IaaS Azure VM instances directly to SQL Server 2022 IaaS Azure VMs exists in the current Microsoft official documentation or provided sources. Therefore, this document consolidates best practices, known generic upgrade guidance, critical components considerations, and validation steps based on the authoritative references listed above only.

---

## Table of Contents

1. [Overview](#overview)  
2. [Preparation and Prerequisites](#preparation-and-prerequisites)  
3. [Migration Challenges and Risks](#migration-challenges-and-risks)  
4. [Migration Strategy Options](#migration-strategy-options)  
5. [Pre-Migration Steps](#pre-migration-steps)  
6. [Backup and Validation](#backup-and-validation)  
7. [Login and Security Transfer](#login-and-security-transfer)  
8. [Agent Jobs Migration](#agent-jobs-migration)  
9. [Linked Servers Reconfiguration](#linked-servers-reconfiguration)  
10. [Database Restore and Upgrade](#database-restore-and-upgrade)  
11. [Post-Migration Validation](#post-migration-validation)  
12. [Troubleshooting](#troubleshooting)  
13. [Rollback Procedures](#rollback-procedures)  
14. [Monitoring and Performance Tuning](#monitoring-and-performance-tuning)  

---

## 1. Overview

This guide is designed for senior database architects and system engineers tasked with migrating SQL Server instances hosted on Azure Infrastructure as a Service (IaaS) virtual machines from version 2019 to 2022. It addresses key aspects of the upgrade process including security principals, job management, linked server configuration, backup and restore operations, and validation to ensure production readiness.

> **Reminder:** This is a generic framework and must be adapted with testing and further refinement according to your environment.

---

## 2. Preparation and Prerequisites

### 2.1 Verify Environment Compatibility

| Component                        | SQL Server 2019 IaaS VM                                | SQL Server 2022 IaaS VM                                | Notes                              |
|---------------------------------|-------------------------------------------------------|-------------------------------------------------------|-----------------------------------|
| Operating System                | Windows Server 2016/2019, supported by SQL 2019        | Windows Server 2019/2022, supported by SQL 2022        | OS upgrade may be necessary        |
| SQL Server Edition             | Enterprise / Standard / Developer                        | Same as source or upgraded edition                     | Check licensing                   |
| VM Size                       | According to workload                                    | Must meet SQL Server 2022 resource requirements        | Validate CPU, RAM sizes             |
| Network Configuration         | Azure VNet, NSG configured for SQL traffic              | Same configuration                                     | Ensure firewall/NSGs consistency   |

### 2.2 Team and Access Preparation

- Ensure all required personnel have administrative access to both source and target VMs.
- Active Directory and domain join status must be confirmed.
- Enable logging and audit trails to monitor each step.
- Verify latest updates and patches are installed for OS and SQL Server.

### 2.3 Required Tools

- SQL Server Management Studio (SSMS), latest version recommended.
- Azure PowerShell and Azure CLI installed on admin machine.
- Backup and restore tools (native SQL backup, Azure Backup if used).
- Script execution environment (PowerShell, SQLCMD).

---

## 3. Migration Challenges and Risks

Documented critical risks to address during migration:

| Risk                                | Impact                                             | Mitigation/Notes                                                              |
|-----------------------------------|----------------------------------------------------|-------------------------------------------------------------------------------|
| Orphaned Users due to SID mismatch | Security breaches, application failures            | Use SID transfer scripts, re-map users (see Section 7)                       |
| Loss of Agent Jobs                | Scheduled jobs stop running leading to business impact | Script jobs out before migration, re-import jobs on target (see Section 8)    |
| Linked Servers lost or misconfigured | Application failures, cross-server query breakdown | Backup and script Linked Server configs, re-deploy on target (Section 9)     |
| Data Integrity and Downtime       | Data corruption or unacceptable downtime           | Backup validation, Set maintenance windows, use transactionally consistent backups |
| Encryption Keys migration          | Database encryption failures                         | Backup and restore encryption keys securely                                  |

---

## 4. Migration Strategy Options

Since no direct Azure VM upgrade scripts exist, consider below general strategies:

- **In-Place Upgrade on Same VM:** Upgrade SQL Server 2019 to 2022 on the same Azure VM (riskier, requires downtime).
- **Side-By-Side Migration to New VM:** Provision a new Azure VM with SQL Server 2022, migrate databases and configurations from source VM.
- **Backup/Restore Migration:** Backup databases on SQL Server 2019, restore on SQL Server 2022 instance after setup.
- **Azure Database Migration Service (DMS):** For migrations targeting Azure SQL or Managed Instances, DMS exists but does not cover VM-to-VM upgrades.

---

## 5. Pre-Migration Steps

### 5.1 Inventory Current Environment

```sql
-- List all SQL Server logins
SELECT name, SID, type_desc FROM sys.server_principals WHERE type_desc IN ('SQL_LOGIN', 'WINDOWS_LOGIN');

-- List Agent Jobs
EXEC msdb.dbo.sp_help_job;

-- List linked servers
EXEC sp_linkedservers;
```

> **Expected output:** Complete list of logins with SIDs, agent jobs, and linked servers.

### 5.2 Script Out Jobs and Linked Servers

Using SSMS, generate scripts for all Agent Jobs and Linked Servers configurations for later deployment.

### 5.3 Backup System Databases (master, msdb, model)

These contain logins, jobs, and server-level configuration.

```sql
-- Take SQL Server backup of master and msdb databases
BACKUP DATABASE master TO DISK = N'C:\Backup\master.bak';
BACKUP DATABASE msdb TO DISK = N'C:\Backup\msdb.bak';
```

---

## 6. Backup and Validation

### 6.1 Full Database Backup

Perform full backups of all user and system databases before migration.

```sql
-- Backup all user databases (example for one database)
BACKUP DATABASE [MyDB] TO DISK = N'C:\Backup\MyDB.bak' WITH CHECKSUM;
```

### 6.2 Validate Backups with DBCC CHECKSUM

```sql
DBCC CHECKDB ('MyDB') WITH NO_INFOMSGS, ALL_ERRORMSGS;
```

> **Expected output:** No errors. Any errors must be addressed before proceeding.

---

## 7. Login and Security Transfer

### 7.1 Capture SQL Logins and SIDs

Scripts for login extraction including hashed passwords are critical to prevent orphaned users.

Microsoft reference: [Transfer Logins and Passwords](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/security/transfer-logins-passwords-between-instances)

> ⚠️ Direct commands not provided here due to source limitations; use Microsoft's scripts from official docs.

### 7.2 Resolve Orphaned Users

After migration, run:

```sql
EXEC sp_change_users_login 'Report';
-- To fix orphaned user:
EXEC sp_change_users_login 'Auto_Fix', 'user_name';
```

Reference: [Troubleshoot Orphaned Users](https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/troubleshoot-orphaned-users-sql-server)

---

## 8. Agent Jobs Migration

1. Script all jobs using SSMS or T-SQL (`msdb.dbo.sp_help_job`).
2. Deploy scripts on the target SQL Server 2022 instance.
3. Validate each job runs correctly; check job owners' logins transferred as per Section 7.

Reference: [SQL Server Agent Jobs](https://learn.microsoft.com/en-us/sql/ssms/agent/sql-server-agent)

---

## 9. Linked Servers Reconfiguration

1. Script linked servers on the source instance:

```sql
EXEC sp_helpserver; -- list servers
```

2. Use scripted creation commands to redeploy on target:

Reference: [Linked Servers Configuration](https://learn.microsoft.com/en-us/sql/relational-databases/linked-servers/linked-servers-database-engine)

---

## 10. Database Restore and Upgrade

On the new VM with SQL Server 2022 installed:

1. Restore system databases if necessary (master, msdb).  
2. Restore user databases from backups.

```sql
RESTORE DATABASE [MyDB] FROM DISK = N'C:\Backup\MyDB.bak' WITH RECOVERY;
```

> **Note:** Restoring system databases from different versions must be handled carefully—often better to recreate logins/jobs via scripts.

3. Run the SQL Server Upgrade Advisor on restored databases to detect deprecated features.

---

## 11. Post-Migration Validation

- Validate database compatibility levels.
- Confirm all logins map correctly to database users.
- Run `DBCC CHECKDB` on all databases.
- Test all Agent Jobs manually.
- Confirm linked servers respond correctly.
- Check SQL Server error logs for warnings/errors.
- Perform application connectivity testing.

---

## 12. Troubleshooting

| Symptom                       | Possible Cause                  | Remedy                                                       |
|-------------------------------|--------------------------------|--------------------------------------------------------------|
| Orphaned users errors         | SID mismatch                   | Use `sp_change_users_login` to fix mapping                  |
| Agent jobs don’t run          | Invalid job owner or missing login | Validate login mapping, fix with scripts                    |
| Failed linked server queries  | Missing or misconfigured linked server | Recreate linked server with correct security context        |
| Backup restore failures       | Backup corrupted or version incompatibility | Verify backup checksum, consider alternate backup            |

---

## 13. Rollback Procedures

- Maintain full backups before migration.
- If issues occur post-migration, restore backups to original environment.
- Revert DNS or connection strings to point back to original environment.
- Document changes thoroughly to aid rollback.

---

## 14. Monitoring and Performance Tuning

- Use SQL Server Performance Monitor counters and Query Store.
- Monitor error logs continually.
- Tune indexes and statistics post-upgrade.
- Review new features and adapt workloads accordingly.

---

# Appendix: ASCII Diagram of Migration Scenario

```text
+----------------------+                                 +----------------------+
| Azure VM SQL Server   |                                 | Azure VM SQL Server   |
| 2019 Instance        |  --- Backup & Scripts -->       | 2022 Instance        |
|                      |                                 |                      |
| - Databases          |                                 | - Restored Databases |
| - Logins (SIDs)      |                                 | - Mapped Logins      |
| - Agent Jobs         |                                 | - Imported Jobs      |
| - Linked Servers     |                                 | - Configured Linked  |
+----------------------+                                 +----------------------+
           |                                                      |
           +----------------- Migration Process ------------------+
```

---

## References and Links

- [Azure DMS: SQL Server to Azure SQL Database](https://learn.microsoft.com/en-us/azure/dms/tutorial-sql-server-to-azure-sql)  
- [Transfer Logins and Passwords Between SQL Server Instances](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/security/transfer-logins-passwords-between-instances)  
- [Troubleshoot Orphaned Users](https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/troubleshoot-orphaned-users-sql-server)  
- [SQL Server Agent Jobs](https://learn.microsoft.com/en-us/sql/ssms/agent/sql-server-agent)  
- [Linked Servers Configuration](https://learn.microsoft.com/en-us/sql/relational-databases/linked-servers/linked-servers-database-engine)

---

> **📢 Disclaimer:** This guide consolidates best practices and official guidance where available, but due to lack of direct Microsoft-provided full procedures for Azure IaaS VM SQL Server version upgrades, you must perform thorough testing and validation within your environment. Always consult the latest official documentation.

---

# END OF DOCUMENT

---

## 📊 Génération

- **Généré**: 12/15/2025, 5:03:04 PM
- **Langue**: English
- **Modèle**: Perplexity Sonar
- **Score audit**: 90/100
- **Statut**: APPROVED
