<!--
Métadonnées du document (invisibles dans le rendu)
title: Guide: Migrate oracle 19c to PostgreSQL
description: migrate oracle 19c to PostgreSQL
keywords: oracle, postgresql
category: sql
subcategory: 
document_type: guide
language: en
language_name: English
version: 1.0
status: production
generated_date: 2025-12-15T17:10:50.869Z
data_source: 
data_composition: official_only
enriched: 
sources_count: 1
direct_links_used: 
verification_status: verified
-->

# Guide: Migrate oracle 19c to PostgreSQL

![Oracle](https://img.shields.io/badge/Oracle-F80000?style=for-the-badge&logo=oracle&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Lang](https://img.shields.io/badge/Lang-EN-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production-green?style=for-the-badge)
![Data](https://img.shields.io/badge/Data-100%25-success?style=for-the-badge)

---

## 📋 Document Metadata
### Critical Links (to reference in documentation):
- Ora2Pg: Oracle to PostgreSQL Migration Tool: [https://ora2pg.darold.net/documentation.html](https://ora2pg.darold.net/documentation.html) (Category: oracle-postgresql)

### Validation Checklist Keywords (must appear in this document):
login, SID, orphaned, user, permission, GRANT, job, agent, linked, server, validation, checksum, DBCC, backup, restore, monitoring, security, authentication, encryption, high availability, disaster recovery, performance, troubleshooting, script

### Known Risks to Address:
1. Data type incompatibilities and semantic differences between Oracle and PostgreSQL  
2. Migration and synchronization of users, roles, and permissions including handling of Oracle SIDs and PostgreSQL roles  
3. Conversion of PL/SQL code, stored procedures, triggers, and functions to PL/pgSQL or alternatives  
4. Handling large data volumes with minimal downtime and ensuring data integrity post-migration  
5. Re-implementing Oracle-specific features such as sequences, partitioning, and advanced indexing in PostgreSQL  

---

# Table of Contents

1. [Introduction and Overview](#1-introduction-and-overview)  
2. [Prerequisites and Environment Preparation](#2-prerequisites-and-environment-preparation)  
3. [Installing Ora2Pg Migration Tool](#3-installing-ora2pg-migration-tool)  
4. [Configuring Ora2Pg](#4-configuring-ora2pg)  
5. [Schema Extraction and Conversion](#5-schema-extraction-and-conversion)  
6. [Data Migration](#6-data-migration)  
7. [PL/SQL to PL/pgSQL Migration](#7-plsql-to-plpgsql-migration)  
8. [User, Roles, and Permission Migration](#8-user-roles-and-permission-migration)  
9. [Advanced Migration Topics and Oracle Features](#9-advanced-migration-topics-and-oracle-features)  
10. [Validation, Testing, and Troubleshooting](#10-validation-testing-and-troubleshooting)  
11. [Rollback and Disaster Recovery](#11-rollback-and-disaster-recovery)  
12. [Performance Optimization and Monitoring](#12-performance-optimization-and-monitoring)  
13. [Appendices](#13-appendices)  

---

## 1. Introduction and Overview

Migration from Oracle 19c to PostgreSQL is a complex, multi-phase process involving schema conversion, data migration, code transformation, and user/permission synchronization. This guide provides an enterprise-level, step-by-step approach using **Ora2Pg** — a reliable open-source tool designed specifically for migrating Oracle databases to PostgreSQL.  

The target audience is senior database architects and DBAs experienced in Oracle and PostgreSQL environments. The document covers detailed technical commands, configurations, error handling, and risk mitigation strategies required for a fully production-ready migration.

---

## 2. Prerequisites and Environment Preparation

### 2.1 Infrastructure Requirements

- **Oracle 19c** database accessible remotely for migration operations.
- Target **PostgreSQL** instance (version compatible with features and Ora2Pg support).
- Migration host machine running Debian/Ubuntu Linux (for this example).

### 2.2 Network and Access

- Ensure connectivity between migration host and Oracle/PostgreSQL servers.
- Oracle user account with appropriate privileges (at least SELECT on all objects).
- PostgreSQL superuser for schema creation, user, and permission management.

### 2.3 Backup and Disaster Recovery Planning

Before starting migration:

- Take full backup of Oracle 19c database to prevent data loss.  
- Document all critical Oracle logins, SIDs, users, roles, and permission grants.  
- Prepare rollback plan in case migration validation fails.

**Estimated duration**: Depends on database size. Plan at least 1-2 hours for full backup.

---

## 3. Installing Ora2Pg Migration Tool

### 3.1 Installing Dependencies and Ora2Pg (Debian/Ubuntu)

```bash
# Update package list
sudo apt-get update

# Install required system dependencies for Ora2Pg
sudo apt-get install -y perl libdbd-pg-perl libpq-dev postgresql-client \
  dpkg-dev build-essential alien make gcc perl-doc libio-compress-perl \
  libdbi-perl libdbd-pg-perl libpq5 libpq-dev postgresql-server-dev-all \
  libxml-simple-perl libjson-perl libyaml-perl libnet-pathtools-perl \
  libfile-copy-recursive-perl libtemplate-perl uuid-dev

# Download the stable Ora2Pg package (version 24.1 as example)
wget https://github.com/darold/ora2pg/releases/download/24.1/ora2pg_24.1-1_all.deb

# Install Ora2Pg package
sudo dpkg -i ora2pg_24.1-1_all.deb

# Fix dependencies if any missing
sudo apt-get install -f
```

**Expected output** example snippet after installation success:

```
Setting up ora2pg (24.1-1) ...
Processing triggers for man-db (2.9.1) ...
```

### 3.2 Verify Ora2Pg Installation

```bash
ora2pg --version
```

Expected output:

```
ora2pg version 24.1
```

**Error handling**:  
- If command not found, verify package installation, PATH environment variables.  
- Inspect logs under `/var/log/` for package installation errors.

---

## 4. Configuring Ora2Pg

Ora2Pg requires a configuration file to specify source Oracle connection, target PostgreSQL connection, and migration parameters.

### 4.1 Generate Basic Configuration File

```bash
ora2pg --init_project ora2pg_migration_project
```

This creates a directory `ora2pg_migration_project` with `ora2pg.conf`.

### 4.2 Edit `ora2pg.conf` To Setup Connections

Open `ora2pg_migration_project/ora2pg.conf` and set:

```ini
# Oracle connection string (replace dummy values)
ORACLE_DSN dbi:Oracle:host=oracle_host;sid=ORCL19C;port=1521
ORACLE_USER your_oracle_user
ORACLE_PWD your_oracle_password

# PostgreSQL connection string (replace dummy values)
PG_DSN dbi:Pg:dbname=target_db;host=postgres_host;port=5432
PG_USER postgres_user
PG_PWD postgres_password

# Output directory for exported SQL scripts
OUTPUT /path/to/output

# Type of export (schema, data, all)
EXPORT_SCHEMA 1
EXPORT_DATA 1

# Additional parameters...
```

### 4.3 Important Parameters for Migration

| Parameter           | Description                                  | Example Value               |
|---------------------|----------------------------------------------|-----------------------------|
| ORACLE_DSN          | Oracle connection string                       | dbi:Oracle:host=ip;sid=orcl;port=1521 |
| ORACLE_USER         | Oracle username                                | oracle_mig_user             |
| ORACLE_PWD          | Oracle password                                | secretpass                 |
| PG_DSN              | PostgreSQL connection string                   | dbi:Pg:dbname=pgdb;host=pghost;port=5432 |
| PG_USER             | PostgreSQL username                            | pgadmin                    |
| PG_PWD              | PostgreSQL password                            | pgsecret                  |
| OUTPUT              | Directory for SQL export files                  | /migration/scripts         |
| EXPORT_SCHEMA       | 1 to export schema DDL, 0 otherwise            | 1                         |
| EXPORT_DATA         | 1 to export data dumps as COPY commands         | 1                         |
| DATA_LIMIT          | Rows to export; 0 = all                        | 0                         |

---

## 5. Schema Extraction and Conversion

### 5.1 Extract Oracle Schema to PostgreSQL-compatible SQL

```bash
ora2pg -c /path/to/ora2pg.conf -t TABLE -o /path/to/output/schema.sql
```

- `-t TABLE` extracts tables schema  
- `-o` target output file

Output expected: `/path/to/output/schema.sql` containing DDL statements compatible with PostgreSQL.

### 5.2 Extract Other Objects (Views, Sequences, Indexes, Constraints...)

Example for views:

```bash
ora2pg -c /path/to/ora2pg.conf -t VIEW -o /path/to/output/views.sql
```

Repeat for types (`-t TYPE`), sequences (`-t SEQUENCE`), triggers (`-t TRIGGER`), etc.

### 5.3 Full Schema Export

Optionally:

```bash
ora2pg -c /path/to/ora2pg.conf -t ALL -o /path/to/output/all_schema.sql
```

**Error handling**:  
- Check for data type incompatibilities (see section 9).  
- Ora2Pg log file configurable in `ora2pg.conf` helps to debug conversion issues.

### 5.4 Apply Schema to PostgreSQL

```bash
psql -h postgres_host -U pgadmin -d target_db -f /path/to/output/all_schema.sql
```

**Expected output**: commands executed, no errors.

**Error handling**:  
- Look for conflicts like existing objects, syntax errors.  
- Use psql’s `\set ON_ERROR_STOP on` before running to fail immediately on error.

---

## 6. Data Migration

### 6.1 Export Data from Oracle

```bash
ora2pg -c /path/to/ora2pg.conf -t COPY -o /path/to/output/data.sql
```

This exports data as COPY commands optimized for bulk import into PostgreSQL.

### 6.2 Import Data into PostgreSQL

```bash
psql -h postgres_host -U pgadmin -d target_db -f /path/to/output/data.sql
```

Check import status:

- Number of tuples imported per table  
- Errors related to constraints, type mismatches or encoding

**Error handling**:  
- Investigate constraint violations.  
- Temporarily disable triggers during bulk import to improve speed.

---

## 7. PL/SQL to PL/pgSQL Migration

### 7.1 Extract PL/SQL Code

```bash
ora2pg -c /path/to/ora2pg.conf -t FUNCTION -o /path/to/output/functions.sql
ora2pg -c /path/to/ora2pg.conf -t TRIGGER -o /path/to/output/triggers.sql
```

### 7.2 Conversion Notes

- Ora2Pg attempts to translate PL/SQL procedure/function code to PL/pgSQL syntax.  
- Manual review and testing are **mandatory** due to language differences.  

### 7.3 Deploy Converted Code

```bash
psql -h postgres_host -U pgadmin -d target_db -f /path/to/output/functions.sql
psql -h postgres_host -U pgadmin -d target_db -f /path/to/output/triggers.sql
```

---

## 8. User, Roles, and Permission Migration

### 8.1 User and Roles Migration Strategy

- Oracle **users & SIDs** do not map 1:1 with PostgreSQL roles.
- Extract Oracle users and roles:  
  ⚠️ INFORMATION NON DISPONIBLE — no Oracle user export commands detailed here.  
- Create corresponding roles in PostgreSQL with appropriate authentication methods.

### 8.2 Permission Grants Migration

- Extract GRANT statements from Oracle  
- Rewrite for PostgreSQL syntax and semantics  

⚠️ INFORMATION NON DISPONIBLE — explicit commands or scripts to migrate users, permissions, and orphaned users are beyond the scope of provided data.

### 8.3 Verification

- Query PostgreSQL tables: `pg_roles`, `pg_user` to validate created roles.
- Check permissions on key objects using `\dp` in psql.

---

## 9. Advanced Migration Topics and Oracle Feature Mapping

| Oracle Feature          | PostgreSQL Equivalent / Notes               |
|------------------------|---------------------------------------------|
| Sequences              | Directly supported, often requires migration|
| Partitioning           | Support available but syntax differs         |
| Indexing               | General indexes map; bitmap, function indexes require review |
| PL/SQL                 | Converted to PL/pgSQL with manual adjustments|
| Synonyms               | Not supported directly; use views or schema design |
| Advanced Data Types    | Check datatype compatibility carefully       |

---

## 10. Validation, Testing, and Troubleshooting

### 10.1 Validation

- Data consistency: Compare row counts, checksums, hash totals between Oracle and PostgreSQL tables.
- Functional tests for converted PL/pgSQL code.

### 10.2 Troubleshooting

- Enable detailed logging in Ora2Pg with `LOG_LEVEL=DEBUG` in `ora2pg.conf`.  
- Check errors line-by-line; focus on data type mismatches and constraint violations.

---

## 11. Rollback and Disaster Recovery

### 11.1 Rollback Plan

- Use Oracle backups to restore source if migration fails.  
- Keep PostgreSQL dump files from partial data imports for selective rollback.

---

## 12. Performance Optimization and Monitoring

- Analyze PostgreSQL query plans after migration to optimize indexes.  
- Setup monitoring tools in PostgreSQL (extensions like pg_stat_statements).

---

## 13. Appendices

### 13.1 Example ora2pg.conf Snippet

```ini
ORACLE_DSN dbi:Oracle:host=192.168.1.100;sid=ORCL19C;port=1521
ORACLE_USER mig_user
ORACLE_PWD mypassword

PG_DSN dbi:Pg:dbname=migratedb;host=192.168.1.110;port=5432
PG_USER pgadmin
PG_PWD pgpassword

OUTPUT /var/migration/ora2pg_output
EXPORT_SCHEMA 1
EXPORT_DATA 1
DATA_LIMIT 0
LOG_LEVEL DEBUG
```

### 13.2 Example Schema Export Command with Validation

```bash
set -e  # Stop script on error

ora2pg -c /var/migration/ora2pg_output/ora2pg.conf -t ALL -o /var/migration/ora2pg_output/all_schema.sql

if [ $? -ne 0 ]; then
  echo "Schema export failed."
  exit 1
fi

psql -h 192.168.1.110 -U pgadmin -d migratedb -f /var/migration/ora2pg_output/all_schema.sql

if [ $? -ne 0 ]; then
  echo "Applying schema to PostgreSQL failed."
  exit 1
fi

echo "Schema migration successful."
```

---

> **References**  
> - Ora2Pg Official Documentation: [https://ora2pg.darold.net/documentation.html#install-debian](https://ora2pg.darold.net/documentation.html#install-debian)

---

_End of document._

---

## 📊 Génération

- **Généré**: 12/15/2025, 5:11:29 PM
- **Langue**: English
- **Modèle**: Perplexity Sonar
- **Score audit**: 90/100
- **Statut**: APPROVED
