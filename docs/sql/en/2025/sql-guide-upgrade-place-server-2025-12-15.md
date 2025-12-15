<!--
Métadonnées du document (invisibles dans le rendu)
title: Guide: Upgrade in place SQL Server 2019
description: upgrade in place SQL Server 2019
keywords: sql server, upgrade, in-place upgrade, SQL Server 2019, SQL Server upgrade best practices, SQL Server architecture, SQL Server configuration
category: sql
subcategory: upgrade
document_type: guide
language: en
language_name: English
version: 1.0
status: production
generated_date: 2024-06-05
data_source: official_only
data_composition: official_only
enriched: false
sources_count: 10
direct_links_used: 
verification_status: verified
-->

# Guide: Upgrade in place SQL Server 2019

![Lang](https://img.shields.io/badge/Lang-EN-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production-green?style=for-the-badge)
![Data](https://img.shields.io/badge/Data-100%25-success?style=for-the-badge)

---

## 📋 Metadata

> **Title**: Guide: Upgrade in place SQL Server 2019  
> **Description**: Detailed, production-ready plan for performing an in-place upgrade of SQL Server 2019 instances to a supported newer SQL Server version on Windows, including architecture sizing, prerequisites, configuration, testing, rollback, and monitoring.  
> **Audience**: Senior architects, DBAs, and infrastructure engineers responsible for SQL Server upgrades in production enterprise environments.  
> **Version**: 1.0  
> **Date**: 2024-06-05  

---

## Table of Contents

1. [Introduction](#1-introduction)  
2. [Prerequisites and Preparation](#2-prerequisites-and-preparation)  
3. [Architecture and Sizing Considerations](#3-architecture-and-sizing-considerations)  
4. [Upgrade Process Overview](#4-upgrade-process-overview)  
5. [Detailed Upgrade Steps](#5-detailed-upgrade-steps)  
6. [Post-Upgrade Configuration and Tuning](#6-post-upgrade-configuration-and-tuning)  
7. [Testing and Validation](#7-testing-and-validation)  
8. [Monitoring](#8-monitoring)  
9. [Troubleshooting and Common Issues](#9-troubleshooting-and-common-issues)  
10. [Rollback and Contingency](#10-rollback-and-contingency)  
11. [Appendices](#11-appendices)  

---

## 1. Introduction

This guide provides a comprehensive, step-by-step, production-ready in-place upgrade plan for upgrading Microsoft SQL Server 2019 instances on Windows to a more recent supported SQL Server version.

The document covers the entire lifecycle including prerequisites, architecture, planning, command-level instructions, post-upgrade tuning, monitoring, rollback, and testing. The upgrade addressed here is an in-place upgrade, preserving existing disk structure and instance configuration to minimize migration effort and downtime.

> > ⚠️ Supported in-place upgrades of SQL Server 2019 (15.x) to newer versions are aligned with Microsoft's official upgrade matrices. See official Microsoft docs for exact supported paths:  
> > https://learn.microsoft.com/en-us/sql/database-engine/install-windows/deprecated-and-discontinued-features-in-sql-server-2022?view=sql-server-ver16#upgrade-considerations-and-discontinued-features  
> > https://learn.microsoft.com/en-us/sql/database-engine/install-windows/supported-version-and-edition-upgrades?view=sql-server-ver16

---

## 2. Prerequisites and Preparation

### 2.1 Supported Upgrade Paths

| Current Version | Supported Upgrade Target Version(s)         | Notes                                |
|-----------------|---------------------------------------------|-------------------------------------|
| SQL Server 2019 | SQL Server 2022 (and later supported builds)| Verify edition and feature support  |

> > ⚠️ Always check official documentation to confirm that your specific edition and usage scenarios are supported for in-place upgrade:  
> > https://learn.microsoft.com/en-us/sql/database-engine/install-windows/supported-version-and-edition-upgrades?view=sql-server-ver16

### 2.2 Hardware and OS Requirements

- Ensure the server OS version supports the target SQL Server version (Windows Server 2019, 2022 etc.)
- Validate memory and CPU requirements per target SQL Server version (consult official supported hardware reference)  
- Disk space must be sufficient for upgrade temporary files, rollback logs, and new binaries. Estimate a minimum of 20–30% free disk space on the system and database volumes before upgrade.

### 2.3 Backup Strategy

- Full backups of all user and system databases including system databases (master, model, msdb)  
- Backup all SQL Server Agent jobs, linked servers, SSIS packages, logins, and certificates  
- Take snapshots or VM backups if virtualized  
- Confirm backup integrity by restoring to a test environment

### 2.4 Permissions and Security

- Upgrade must be performed by a Windows user with:

  - Local Administrator permissions  
  - SQL Server sysadmin role credentials  

- Security policies (firewall, antivirus) must allow upgrade process and SQL Server services to start  
- Disable third-party monitoring or antivirus prior to upgrade to avoid interference

### 2.5 Software Prerequisites

- .NET Framework version required by target SQL Server version must be installed  
- Windows updates fully current  
- Install latest SQL Server Setup support files by running the setup executable before upgrade begins  

### 2.6 Instance Inventory and Audit

- Document current instance configuration:

  - Instance name and edition  
  - Service pack and cumulative update level  
  - Server properties and current setup parameters  
  - Linked servers, endpoints, operator alerts, etc.  
  - Version and collation of all databases  

- Run `sp_Blitz` or similar tools to identify potential upgrade blockers or deprecated features used.

---

## 3. Architecture and Sizing Considerations

### 3.1 Instance Architecture Diagram (Example)

```
+---------------------------------------------+
|             SQL Server Instance              |
|                                             |
| +------------------+   +------------------+ |
| | Data Volume (D:)  |   | Log Volume (L:)   | |
| +------------------+   +------------------+ |
| +------------------+                          |
| | TempDB Volume (T:)|                          |
| +------------------+                          |
+---------------------------------------------+

– OS: Windows Server 2019 Standard  
– Memory: 128 GB RAM  
– CPU: 24 cores, 3.2 GHz  
– Storage: SSD enterprise-class with 1 ms latency  
```

### 3.2 Sizing Guidelines

- Ensure tempdb is on a dedicated fast volume with separate physical disks if possible  
- Memory reservation according to workload (dedicate at least 80% of total RAM to SQL Server)  
- Configure max server memory (`sp_configure`) based on total server RAM and other apps

### 3.3 High Availability (HA) and Disaster Recovery (DR)

- If running Always On Availability Groups (AG), ensure AG supports new SQL Server version prior to upgrade  
- Upgrade AG nodes one by one using rolling upgrade procedure — this guide focuses on standalone instances  
- Backup AG configurations and health prior to upgrade  

---

## 4. Upgrade Process Overview

The overall upgrade process includes the following principal phases:

| Step Number | Description                                 | Estimated Duration  |
|-------------|---------------------------------------------|--------------------|
| 1.0         | Pre-upgrade validation and backups          | 1–4 hours          |
| 2.0         | Preparation & prerequisite checks           | 30 mins            |
| 3.0         | Launch in-place upgrade                      | 1–3 hours          |
| 4.0         | Post-upgrade configuration & tuning         | 1–2 hours          |
| 5.0         | Validation, testing, and monitoring setup   | 2–4 hours          |

> > *Note: Durations highly dependent on instance size, environment complexity, and backup speed.*

---

## 5. Detailed Upgrade Steps

### 5.1 Step 1.0 - Pre-upgrade Validation

```powershell
# Validate current SQL Server version
sqlcmd -Q "SELECT @@VERSION"
```

- **Expected Output:** Version string starting with Microsoft SQL Server 2019  
- **If mismatched:** Verify instance targeted for upgrade

### 5.2 Step 1.1 - Full Backup of All Databases

```powershell
# Example T-SQL to backup all user databases dynamically
sqlcmd -S <ServerName> -Q "
DECLARE @name VARCHAR(50) -- database name
DECLARE @path VARCHAR(256) -- path for backup files
DECLARE @fileName VARCHAR(256) -- filename for backup
DECLARE @backupDate VARCHAR(20) -- backup date

SET @path = 'D:\SQLBackups\'
SET @backupDate = CONVERT(VARCHAR(20), GETDATE(), 112)

DECLARE db_cursor CURSOR FOR   
SELECT name FROM sys.databases  
WHERE name NOT IN ('tempdb') AND state = 0 -- online databases

OPEN db_cursor   
FETCH NEXT FROM db_cursor INTO @name   

WHILE @@FETCH_STATUS = 0   
BEGIN   
    SET @fileName = @path + @name + '_' + @backupDate + '.bak'  
    BACKUP DATABASE @name TO DISK = @fileName WITH INIT, COMPRESSION;  
    FETCH NEXT FROM db_cursor INTO @name   
END   

CLOSE db_cursor   
DEALLOCATE db_cursor"
```

- **Check:** Backup files created at specified path  
- **Error Handling:** Abort upgrade if backup fails

### 5.3 Step 1.2 - Backup System Databases

```powershell
# Backup system databases individually as necessary
sqlcmd -Q "BACKUP DATABASE master TO DISK='D:\SQLBackups\master.bak' WITH INIT, COMPRESSION;"
sqlcmd -Q "BACKUP DATABASE msdb TO DISK='D:\SQLBackups\msdb.bak' WITH INIT, COMPRESSION;"
sqlcmd -Q "BACKUP DATABASE model TO DISK='D:\SQLBackups\model.bak' WITH INIT, COMPRESSION;"
```

- Validate successful creation and check file properties

### 5.4 Step 1.3 - Export Logins and Jobs

```powershell
# Export logins with script 'sp_help_revlogin' or official method
-- Reference script: https://support.microsoft.com/en-us/help/918992/how-to-transfer-logins-and-passwords-between-instances-of-sql-server
```

- Save SQL Agent jobs via Management Studio or `msdb.dbo.sp_help_job`

### 5.5 Step 2.0 - Close Connections and Stop SQL Agent Jobs

```powershell
sqlcmd -Q "
ALTER DATABASE tempdb SET RESTRICTED_USER WITH ROLLBACK IMMEDIATE;
ALTER DATABASE <user_db> SET SINGLE_USER WITH ROLLBACK IMMEDIATE;"
```

- Stop SQL Server Agent service:  
```powershell
Stop-Service -Name SQLSERVERAGENT
```

---

### 5.6 Step 3.0 - Run Setup for In-place Upgrade

Run `setup.exe` from the installation media for the target SQL Server version with the following parameters:

```powershell
.\setup.exe /QUIET /ACTION=Upgrade /INSTANCENAME=<InstanceName> /UpdateEnabled=True /IAcceptSQLServerLicenseTerms /INDICATEPROGRESS
```

- `/QUIET`: Runs without UI interaction  
- `/ACTION=Upgrade`: Specifies upgrade action  
- `/INSTANCENAME`: Target instance name (default or named)  
- `/UpdateEnabled`: Allows setup to apply latest updates during upgrade  
- `/IAcceptSQLServerLicenseTerms`: Required to accept license  
- `/INDICATEPROGRESS`: Shows progress in logs

> > ⚠️ The full list of parameters is documented here:  
> > https://learn.microsoft.com/en-us/sql/database-engine/install-windows/command-prompt-sql-server-setup?view=sql-server-ver16

### 5.7 Step 3.1 - Monitor Upgrade Progress

- Tail SQL Server setup log located at `%programfiles%\Microsoft SQL Server\<version>\Setup Bootstrap\Log\Summary.txt`  
- Check for errors or warnings

---

### 5.8 Step 4.0 - Post-Upgrade Configuration

- Verify SQL Server version:

```powershell
sqlcmd -Q "SELECT @@VERSION"
```

- Re-enable `max degree of parallelism (maxdop)`, `cost threshold for parallelism` and other tunings based on workload  
- Run DBCC CHECKDB on all user databases to confirm integrity post-upgrade:

```powershell
sqlcmd -Q "
DECLARE db_cursor CURSOR FOR SELECT name FROM sys.databases WHERE name NOT IN ('tempdb')

OPEN db_cursor
DECLARE @name NVARCHAR(128)

FETCH NEXT FROM db_cursor INTO @name

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT 'Running DBCC CHECKDB on database: ' + @name
    EXEC('DBCC CHECKDB(''' + @name + ''') WITH NO_INFOMSGS')
    FETCH NEXT FROM db_cursor INTO @name
END

CLOSE db_cursor
DEALLOCATE db_cursor
"
```

- Check and update statistics if needed:

```powershell
sqlcmd -Q "EXEC sp_updatestats"
```

---

### 5.9 Step 4.1 - Restart Services

```powershell
Restart-Service -Name MSSQL$<InstanceName>
Restart-Service -Name SQLSERVERAGENT
```

Verify correct start and no errors in SQL Server error logs.

---

### 5.10 Step 5.0 - Validation and Testing

- Run all application test suites and performance benchmarks  
- Validate connectivity and database functionality  
- Monitor SQL error logs for anomalies  
- Check for deprecated or discontinued feature warnings in upgrade report and logs

---

## 6. Post-Upgrade Configuration and Tuning

- Review compatibility level of databases, optionally set to recommended levels:  

```sql
ALTER DATABASE <db_name> SET COMPATIBILITY_LEVEL = 160; -- For SQL Server 2022
```

- This may improve query optimizer efficiency but must be validated in test environment first.

- Review and configure new features available in target SQL Server version.

- Adjust max server memory if workload changed.

- Enable lightweight query profiling for diagnostics if needed.

---

## 7. Testing and Validation

- Run full regression testing on critical applications  
- Confirm backup and restore operations works post-upgrade  
- Monitor performance counters and query execution plans for regressions

---

## 8. Monitoring

- Use SQL Server Management Studio (SSMS) or Azure Data Studio monitoring tools  
- Implement extended events for system health tracking  
- Monitor wait statistics and error logs daily for the first week after upgrade  
- Integrate with existing monitoring solution (e.g., System Center, SCOM, Prometheus, etc.)

---

## 9. Troubleshooting and Common Issues

| Issue                             | Possible Cause                     | Troubleshooting Steps                                         |
|----------------------------------|----------------------------------|---------------------------------------------------------------|
| Upgrade stops with error code 0x| Permission issues or corrupted installer files | Re-run setup as local admin, verify installer checksum          |
| SQL Server fails to start        | Service account permission issue | Verify service account permissions, check event viewer         |
| Database in Suspect or Recovery Pending state | Corrupt database files or incomplete upgrade  | Restore from backup, run DBCC CHECKDB, review upgrade logs     |
| Performance regression            | Default configurations changed   | Tune server parameters, review query plans, check statistics   |

---

## 10. Rollback and Contingency

- If upgrade fails or produces critical issues, rollback strategy:

  - Restore all databases from full backups made pre-upgrade  
  - Reinstall old SQL Server version if binaries were replaced  
  - Restore master database carefully to avoid login mismatch  
  - RESTORE DATABASE commands example:

```sql
RESTORE DATABASE <db_name> FROM DISK = 'D:\SQLBackups\<db_name>_YYYYMMDD.bak' WITH REPLACE;
```

- Verify system stability before permitting production traffic.

---

## 11. Appendices

### 11.1 Useful Links and Documentation

- Supported Upgrade Paths:  
  https://learn.microsoft.com/en-us/sql/database-engine/install-windows/supported-version-and-edition-upgrades?view=sql-server-ver16

- Deprecated Features:  
  https://learn.microsoft.com/en-us/sql/database-engine/install-windows/deprecated-and-discontinued-features-in-sql-server-2022?view=sql-server-ver16

- Command-Line Setup Parameters:  
  https://learn.microsoft.com/en-us/sql/database-engine/install-windows/command-prompt-sql-server-setup?view=sql-server-ver16

- Backup and Restore Documentation:  
  https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/sql-server-backup-and-restore?view=sql-server-ver16

- Troubleshooting Upgrade Issues:  
  https://learn.microsoft.com/en-us/sql/database-engine/install-windows/troubleshooting-installation-errors?view=sql-server-ver16

---

# Summary

This guide provides a thorough, production-ready approach to performing an in-place upgrade from SQL Server 2019 to a supported newer version on Windows. Following this plan and thoroughly testing will help ensure a smooth transition with minimal downtime and risk.

---

---

## 📊 Génération

- **Généré**: 12/15/2025, 4:20:39 PM
- **Langue**: English
- **Modèle**: Perplexity Sonar
- **Score audit**: 90/100
- **Statut**: APPROVED
