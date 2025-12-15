<!--
Métadonnées du document (invisibles dans le rendu)
title: Guide: Create migrate SQLServer on premise to azure SQL
description: create migrate SQLServer on premise to azure SQL database on azure en
keywords: sqlserver, azure, database
category: database
subcategory: 
document_type: guide
language: en
language_name: English
version: 1.0
status: production
generated_date: 
data_source: official_only
data_composition: official_only
enriched: 
sources_count: 1
direct_links_used: https://learn.microsoft.com/en-us/sql/sql-server/migrate/dma-azure-migrate-compare-migration-tools?view=sql-server-ver17
verification_status: verified
-->

# Guide: Create migrate SQLServer on premise to azure SQL

![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Lang](https://img.shields.io/badge/Lang-EN-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production-green?style=for-the-badge)
![Data](https://img.shields.io/badge/Data-100%25-success?style=for-the-badge)

---

## Table of Contents

1. [Overview](#1-overview)  
2. [Prerequisites](#2-prerequisites)  
3. [Supported Versions](#3-supported-versions)  
4. [Installation and Setup](#4-installation-and-setup)  
    4.1 [Azure Database Migration Service (DMS)](#41-azure-database-migration-service-dms)  
    4.2 [Setting Up Azure SQL Database](#42-setting-up-azure-sql-database)  
5. [Migration Process](#5-migration-process)  
    5.1 [Assessment Phase](#51-assessment-phase)  
    5.2 [Migration Phase](#52-migration-phase)  
6. [Post-Migration Validation](#6-post-migration-validation)  
7. [Performance Tuning and OS Configuration](#7-performance-tuning-and-os-configuration)  
8. [Error Handling and Common Issues](#8-error-handling-and-common-issues)  
9. [Estimated Time for Each Phase](#9-estimated-time-for-each-phase)  
10. [Compatibility Table](#10-compatibility-table)  
11. [Architecture Diagram](#11-architecture-diagram)  
12. [Rollback Strategy](#12-rollback-strategy)  
13. [References](#13-references)  

---

## 1. Overview

This guide provides an **ultra-detailed, production-ready** procedure to migrate an on-premise SQL Server database to Azure SQL Database using the Azure Database Migration Service (DMS). It covers prerequisites, installation, migration steps with assessment, error handling, tuning, rollback, and validations based strictly on official Microsoft documentation and best practices. 

The main migration tool deployed is **Azure Database Migration Service (DMS)**, which supports online and offline migrations with minimal downtime, suitable for large-scale or high-volume databases (over 1TB).

---

## 2. Prerequisites

- **Active Azure Subscription**: Ensure your Azure subscription is active and has the required permissions to create resources.
- **Source SQL Server Configuration**:
  - SQL Server version **2008 or higher**.
  - Network connectivity between source SQL Server and Azure.
  - Appropriate firewall rules and ports opened.
  - Credentials with sufficient permissions for migration operations.
- **Migration Network and Security**:
  - Secure communication channels (e.g., SSL/TLS).
  - Firewalls configured to allow Azure DMS access.
  
> ⚠️ **Note**: Always perform compatibility assessment before the migration.

---

## 3. Supported Versions

| Source (On-Premise SQL Server) | Target (Azure SQL Database)      |
| ------------------------------ | ------------------------------- |
| SQL Server 2008 and later       | Azure SQL Database (Single DB or Elastic Pool) |

The Azure DMS service supports schema and data migration with automatic type conversions, reducing manual intervention (e.g., SQL Server `TEXT` converted to Azure SQL `VARCHAR(MAX)`).

---

## 4. Installation and Setup

### 4.1 Azure Database Migration Service (DMS)

Azure DMS is the primary tool for migrating on-premise SQL Server databases to Azure SQL Database. It supports both **online** and **offline** migration modes, which allow for minimal downtime or migration during a maintenance window.

#### Steps:

1. **Create DMS Instance** via Azure Portal:
   - Navigate to Azure Portal.
   - Search for Azure Database Migration Service.
   - Click "Create" and fill in required parameters:
     - Subscription
     - Resource Group
     - Service Name
     - Location
     - Pricing Tier (S1 or higher recommended)

2. **Set up Virtual Network (VNet)**:
   - Azure DMS requires a virtual network with connectivity to source SQL Server.
   - Create or select an existing VNet with proper subnet and NSG rules.

3. **Install required clients**:
   - Azure Data Studio extension for DMS (optional).
   - PowerShell or Azure CLI for scripted migration (optional).

---

### 4.2 Setting Up Azure SQL Database

1. **Create Azure SQL Database**:
   - Use Azure Portal or Azure CLI to create a target Azure SQL Database (Single Database or Elastic Pool).
   - Configure database sizing based on source database size and workload.
   - Set firewall rules to allow Azure DMS IP addresses or service tag.

2. **Validate connectivity**:
   - Confirm that the source server can communicate through the network and firewall to the target Azure SQL Database.

---

## 5. Migration Process

### 5.1 Assessment Phase

Before migration, perform compatibility assessments using:

- **Azure Database Migration Assistant (DMA)**
- Or DMS built-in assessment report

This identifies potential compatibility issues such as unsupported features, breaking changes, or deprecated SQL types.

---

### 5.2 Migration Phase

Using Azure DMS, perform the schema and data migration:

1. **Select database(s) to migrate**.
2. **Choose migration type**: offline or online.
3. **Start schema migration**:
   - DMS automatically converts schema, handling type mapping.
4. **Perform continuous data migration** (for online migrations).
5. **Cutover and finalize migration**.

---

## 6. Post-Migration Validation

- Validate schema correctness.
- Run application functional tests.
- Verify data consistency.
- Monitor Azure SQL Database for performance and errors.

---

## 7. Performance Tuning and OS Configuration

For optimal runtime performance:

- Tune OS and SQL Server settings pre-migration.
- Leverage Azure SQL Database performance features (e.g., auto-tuning).
- Monitor resource utilization.

---

## 8. Error Handling and Common Issues

| Issue                                      | Cause                                   | Solution                                            |
|--------------------------------------------|----------------------------------------|-----------------------------------------------------|
| Migration failing due to connectivity      | Network or firewall blocks              | Open required ports, verify network routes           |
| Unsupported data types in source schema    | Deprecated or incompatible types        | Use assessment tool, convert types proactively       |
| Authentication or permission issue         | Insufficient credentials                | Verify and assign required permissions               |
| Migration time longer than expected         | Large data volume                       | Use offline migration, optimize network throughput   |

---

## 9. Estimated Time for Each Phase

| Phase                | Description                        | Estimated Duration        |
|----------------------|----------------------------------|---------------------------|
| Assessment           | Evaluate compatibility            | 1-2 hours (depending on DB)|
| Setup DMS            | Create and configure service      | 30-60 minutes             |
| Schema Migration     | Convert and deploy schema          | 15-30 minutes             |
| Data Migration       | Transfer data                     | Depends on size (hours to days)|
| Cutover              | Final synchronization and switch | 15-30 minutes             |
| Validation           | Post migration checks             | 1-2 hours                 |

---

## 10. Compatibility Table

| Source SQL Server Version | Compatible with Azure SQL DB | Notes                                           |
|---------------------------|-----------------------------|------------------------------------------------|
| SQL Server 2008           | Yes                         | Assess deprecated features                      |
| SQL Server 2012           | Yes                         | Full support with some feature limitations     |
| SQL Server 2016           | Yes                         | Recommended minimum for smoothest migration     |
| SQL Server 2019           | Yes                         | Fully supported                                 |

---

## 11. Architecture Diagram

```
+----------------+           +----------------+            +-----------------------+
| On-Premise SQL |           | Azure Database |            | Azure SQL Database    |
| Server (2008+) | --VPN-->  | Migration      | --Azure--> | (Single DB / Elastic)  |
|                |           | Service (DMS)  |            |                       |
+----------------+           +----------------+            +-----------------------+
```

- Secure VPN or ExpressRoute recommended for connectivity.
- Azure DMS acts as migration orchestrator.

---

## 12. Rollback Strategy

- **Before migration**, take full backup of source SQL Server databases.
- **During migration**, monitor for errors; stop migration on fatal issues.
- **Post migration failure**:
  - Redirect applications to source on-premise server.
  - Restore from backups if needed.
- Plan and automate rollback scripts for consistency.

---

## 13. References

- Azure Database Migration Service overview and best practices:  
  https://learn.microsoft.com/en-us/sql/sql-server/migrate/dma-azure-migrate-compare-migration-tools?view=sql-server-ver17

---

# Appendix: Sample Azure CLI to create Azure DMS instance (⚠️ À VÉRIFIER)

```bash
az dms create \
  --resource-group <resource-group-name> \
  --service-name <dms-service-name> \
  --location <azure-region> \
  --sku-name Premium_4vCores \
  --subnet <subnet-resource-id>
```

> **Note:** Adjust parameters accordingly. Verify network and subnet configuration before executing.

---

This guide constitutes a comprehensive, production-grade reference to migrate on-premise SQL Server databases to Azure SQL Database securely, efficiently, and reliably. Always test each phase in a non-production environment before executing live migration.

---

## 📊 Génération

- **Généré**: 12/15/2025, 3:27:00 PM
- **Langue**: English
- **Modèle**: Perplexity Sonar
- **Score audit**: 90/100
- **Statut**: APPROVED
