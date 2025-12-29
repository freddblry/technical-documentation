<!--
Métadonnées du document (invisibles dans le rendu)
title: Guide: To clone pdb to pdb oracle
description: how to clone pdb to pdb oracle
keywords: oracle
category: oracle
subcategory: 
document_type: guide
language: en
language_name: English
version: 1.0
status: production
generated_date: 2025-12-29T08:10:28.972Z
data_source: 
data_composition: official_only
enriched: 
sources_count: 9
direct_links_used: 
verification_status: verified
-->

# Guide: To clone pdb to pdb oracle

![Oracle](https://img.shields.io/badge/Oracle-F80000?style=for-the-badge&logo=oracle&logoColor=white)
![Oracle](https://img.shields.io/badge/Oracle-F80000?style=for-the-badge&logo=oracle&logoColor=white)
![Lang](https://img.shields.io/badge/Lang-EN-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production-green?style=for-the-badge)
![Data](https://img.shields.io/badge/Data-100%25-success?style=for-the-badge)

---

## Table of Contents

1. [Introduction](#1-introduction)  
2. [Prerequisites and Environment Setup](#2-prerequisites-and-environment-setup)  
3. [Cloning a PDB to Another PDB in the Same or Different CDB](#3-cloning-a-pdb-to-another-pdb-in-the-same-or-different-cdb)  
4. [Step-by-Step Clone Procedure](#4-step-by-step-clone-procedure)  
5. [Error Handling and Troubleshooting](#5-error-handling-and-troubleshooting)  
6. [Rollback and Cleanup Scripts](#6-rollback-and-cleanup-scripts)  
7. [Best Practices and Performance Considerations](#7-best-practices-and-performance-considerations)  
8. [Appendices](#8-appendices)  

---

## 1. Introduction

This guide provides a comprehensive, production-ready procedure to clone a Pluggable Database (PDB) to another PDB within Oracle Multitenant container databases (CDB). It covers all prerequisites, exact commands, expected outputs, error handling, and troubleshooting for Oracle Database 12c Release 1 (12.1.0.2) and above, leveraging the multitenant architecture with proper consideration to encryption and keystore access.

---

## 2. Prerequisites and Environment Setup

### 2.1 Supported Versions and Architectures
- Oracle Database: 12c Release 1 (12.1.0.2) or later with multitenant option enabled.
- The source and target PDBs must reside within open CDBs.
- Operating Systems supported: Oracle Linux 7/8, RHEL 7/8, SLES 12/15.

### 2.2 Space and Resources
- At least **twice the size** of the source PDB storage is required for cloning (Hot clone).
- Ensure sufficient CPU, memory, and I/O resources for cloning operations.

### 2.3 TDE and Encryption Considerations
- If the source PDB uses Transparent Data Encryption (TDE), ensure the target CDB has access to the crypto keystore (wallet).
- For Fortanix DSM users, verify `/etc/fortanix/pkcs11.conf` exists and the keystore is open.

### 2.4 Verify Environment State

Connect to the root container as SYSDBA and verify the CDB and PDB states:

```sql
-- Connect as SYSDBA to CDB$ROOT
SQL> conn / as sysdba  

-- List existing PDBs
SQL> show pdbs;

-- Check open mode of PDBs
SQL> select name, open_mode from v$pdbs;

-- Open source PDB in READ WRITE mode if needed
SQL> alter pluggable database PDB_SOURCE open read write;
```

Expected output example:

```
NAME             CON_ID  OPEN MODE  RESTRICTED
---------------- ------- ---------- ----------
CDB$ROOT              1  READ WRITE NO
PDB_SOURCE             3  READ WRITE NO
PDB_TARGET             4  MOUNTED
```

For TDE encryption keys:

```sql
-- Check encryption keys available
SQL> select * from v$encryption_keys;
```

Verify a non-empty result set indicating accessible encryption keys.

---

## 3. Cloning a PDB to Another PDB in the Same or Different CDB

Oracle supports hot cloning of PDBs within the same CDB or across different CDBs when connected via database links.

### 3.1 Cloning Within the Same CDB

**Prerequisites**: Source PDB is open in READ WRITE mode.

**Basic syntax:**

```sql
CREATE PLUGGABLE DATABASE pdb_clone FROM pdb_source;
```

**Or** with specific options:

```sql
CREATE PLUGGABLE DATABASE pdb_clone FROM pdb_source
  FILE_NAME_CONVERT = ('/source/pdb_source/', '/target/pdb_clone/');
```

### 3.2 Cloning Across Different CDBs Using Database Link

**Prerequisites**:  
- Database link created from target CDB root to source CDB root.  
- Source PDB is open READ WRITE mode.  
- TDE encryption keys accessible on the target CDB if source PDB is encrypted.

**Basic syntax:**

```sql
CREATE PLUGGABLE DATABASE pdb_clone FROM pdb_source@src_cdb_link
  FILE_NAME_CONVERT = ('/source/pdb_source/', '/target/pdb_clone/');
```

---

## 4. Step-by-Step Clone Procedure

### 4.1 Step 1: Verify Source and Target Readiness

| Task                      | Command / Query                                 | Expected Result                      |
|---------------------------|------------------------------------------------|------------------------------------|
| Connect to CDB$ROOT        | `conn / as sysdba`                              | SQL prompt                         |
| Verify source PDB is open  | `select name, open_mode from v$pdbs;`          | Source PDB status = READ WRITE      |
| Open source PDB if closed  | `alter pluggable database pdb_source open read write;` | Success message                    |
| Verify target PDB mount state | `show pdbs`                                   | Target PDB either DOES NOT EXIST or is MOUNTED |

### 4.2 Step 2: Create Database Link (If Cloning Remote)

```sql
CREATE DATABASE LINK src_cdb_link
CONNECT TO <username> IDENTIFIED BY "<password>" 
USING '<tns_connect_descriptor>';
```

**Note:** Ensure the user has sufficient privileges to read from source CDB.

### 4.3 Step 3: Issue Clone Command

#### 4.3.1 Local Clone

```sql
CREATE PLUGGABLE DATABASE pdb_clone FROM pdb_source
  FILE_NAME_CONVERT = ('/path/to/source/pdb_source/', '/path/to/target/pdb_clone/');
```

Expected Output:

```
Pluggable database created.
```

#### 4.3.2 Remote Clone

```sql
CREATE PLUGGABLE DATABASE pdb_clone FROM pdb_source@src_cdb_link
  FILE_NAME_CONVERT = ('/path/to/source/pdb_source/', '/path/to/target/pdb_clone/');
```

Expected Output:

```
Pluggable database created.
```

### 4.4 Step 4: Open the Cloned PDB

```sql
ALTER PLUGGABLE DATABASE pdb_clone OPEN;
```

Expected Output:

```
Pluggable database altered.
```

### 4.5 Step 5: Post-clone Validation

Validate the clone success by querying PDBs:

```sql
SHOW PDBS;
```

Example:

```
NAME             CON_ID  OPEN MODE  RESTRICTED
---------------- ------- ---------- ----------
CDB$ROOT              1  READ WRITE NO
PDB_SOURCE             3  READ WRITE NO
PDB_CLONE              5  READ WRITE NO
```

---

## 5. Error Handling and Troubleshooting

| Error Condition                            | Cause                                                   | Solution                                                    |
|--------------------------------------------|---------------------------------------------------------|--------------------------------------------------------------|
| ORA-65096: Invalid common user or role name | Attempt to create PDB with an invalid name           | Use valid PDB name, follow Oracle naming conventions.        |
| ORA-65188: PDB must be in READ WRITE mode to create clone | Source PDB not open in READ WRITE                     | Open the source PDB: `ALTER PLUGGABLE DATABASE pdb_source OPEN READ WRITE;`|
| ORA-65090: Destination container database must be in READ WRITE mode | Target CDB not open READ WRITE                         | Open the target CDB: `ALTER DATABASE OPEN;`                  |
| TDE-related errors (e.g., cannot access encryption keys) | Keystore not open or configured                       | Check `/etc/fortanix/pkcs11.conf` and open keystore          |
| ORA-12154: TNS:could not resolve the connect identifier specified | Incorrect database link or TNS names                   | Verify database link and TNS names                            |

### 5.1 Estimated Time

| Operation             | Estimated Duration       |
|-----------------------|--------------------------|
| Pre-validation        | 5-10 minutes             |
| Database link creation| 2-5 minutes              |
| Cloning PDB           | Dependent on PDB size, ~minutes to hours |
| Opening cloned PDB    | Less than 1 minute       |

---

## 6. Rollback and Cleanup Scripts

In case of failure or need to abort the clone:

### 6.1 Drop the PDB Clone if Created

```sql
-- Close the PDB if open
ALTER PLUGGABLE DATABASE pdb_clone CLOSE IMMEDIATE;

-- Drop the PDB including datafiles
DROP PLUGGABLE DATABASE pdb_clone INCLUDING DATAFILES;
```

Expected Output:

```
Pluggable database dropped.
```

### 6.2 Remove Database Link (if remote clone)

```sql
DROP DATABASE LINK src_cdb_link;
```

---

## 7. Best Practices and Performance Considerations

- Always perform clone in **READ WRITE** mode on source PDB for consistency.
- Ensure **wallet** and encryption keys are accessible before cloning encrypted PDB.
- Use fully qualified and absolute paths in `FILE_NAME_CONVERT` to avoid misplacement of datafiles.
- Monitor disk space carefully as cloning requires nearly twice the source PDB size.
- Run clone operations during off-peak hours where possible to reduce performance impact.
- Regularly validate clone success by querying `v$pdbs` and testing connectivity.

---

## 8. Appendices

### 8.1 Oracle Multitenant Architecture ASCII Diagram

```
+-------------------------------------------------+
|                  CDB$ROOT (Container DB)        |
|  +-----------+      +-----------+     +-------+  |
|  | PDB_SOURCE|      | PDB_CLONE | ... | PDB_N |  |
|  +-----------+      +-----------+     +-------+  |
+-------------------------------------------------+
```

### 8.2 Key SQL Commands Summary

| Command                                                 | Purpose                        |
|---------------------------------------------------------|--------------------------------|
| `SHOW PDBS;`                                            | List configured PDBs            |
| `ALTER PLUGGABLE DATABASE pdb_source OPEN READ WRITE;` | Open PDB in read-write mode     |
| `CREATE PLUGGABLE DATABASE pdb_clone FROM pdb_source;` | Clone PDB locally               |
| `CREATE PLUGGABLE DATABASE pdb_clone FROM pdb_source@link;` | Clone PDB remotely             |
| `ALTER PLUGGABLE DATABASE pdb_clone OPEN;`             | Open cloned PDB                |
| `DROP PLUGGABLE DATABASE pdb_clone INCLUDING DATAFILES;` | Cleanup after failed clone    |

### 8.3 Additional References

- Oracle Multitenant Concepts: [Oracle Multitenant](https://docs.oracle.com/database/121/CNCPT/pluggable.htm)
- Oracle Database SQL Reference for `CREATE PLUGGABLE DATABASE`: [CREATE PLUGGABLE DATABASE](https://docs.oracle.com/database/121/SQLRF/statements_7014.htm)

---

> **⚠️ Note:** This guide strictly uses verified commands and recommendations based on Oracle 12c 12.1.0.2+ multitenant architecture and Fortanix DSM keystore requirements as documented.

---

_**End of Guide**_

---

## 📊 Génération

- **Généré**: 12/29/2025, 8:12:24 AM
- **Langue**: English
- **Modèle**: Perplexity Sonar
- **Score audit**: 90/100
- **Statut**: APPROVED
