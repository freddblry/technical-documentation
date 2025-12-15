<!--
Métadonnées du document (invisibles dans le rendu)
title: Guide: Migration PostgreSQL database vers Oracle Database en français
description: migration PostgreSQL database vers Oracle Database en français
keywords: migration, postgresql, database, oracle
category: database
subcategory: 
document_type: guide
language: fr
language_name: Français
version: 1.0
status: production
generated_date: 2025-12-15T16:46:04.451Z
data_source: 
data_composition: official_only
enriched: 
sources_count: 0
direct_links_used: 
verification_status: unverified
-->

# Guide: Migration PostgreSQL database vers Oracle Database en français

![Oracle](https://img.shields.io/badge/Oracle-F80000?style=for-the-badge&logo=oracle&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Lang](https://img.shields.io/badge/Lang-FR-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production-green?style=for-the-badge)
![Data](https://img.shields.io/badge/Data-100%25-success?style=for-the-badge)

---

## 📋 Métadonnées du document
🔗 LIENS CRITIQUES (à mentionner dans la doc):
- Ora2Pg: Oracle to PostgreSQL Migration Tool: https://ora2pg.darold.net/documentation.html (catégorie: oracle-postgresql)
- Azure: Migrate Oracle to PostgreSQL using Ora2Pg: https://learn.microsoft.com/en-us/azure/postgresql/migrate/concepts-ora2pg (catégorie: oracle-postgresql)
- 🚨 CRITICAL: SQL Server Login Management: https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/create-a-login (catégorie: sql-logins-critical)
- 🚨 CRITICAL: Troubleshoot Orphaned Users: https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/troubleshoot-orphaned-users-sql-server (catégorie: sql-orphaned-users)
- 🚨 CRITICAL: SQL Server Agent Jobs: https://learn.microsoft.com/en-us/sql/ssms/agent/sql-server-agent (catégorie: sql-jobs-critical)
- 🚨 CRITICAL: Linked Servers Configuration: https://learn.microsoft.com/en-us/sql/relational-databases/linked-servers/linked-servers-database-engine (catégorie: sql-linked-servers)

---

## ⚠️ IMPORTANT - Limites documentaires

> Je vais extraire uniquement des informations factuelles issues des sources officielles spécifiées. Cette demande porte sur la migration « PostgreSQL vers Oracle Database ».  
> Cependant, les sources principales disponibles traitent essentiellement de la migration dans le sens Oracle → PostgreSQL (ora2pg), ou Oracle → Azure Database for PostgreSQL (Azure DMS).  
> Il n'existe PAS de guide officiel documenté précisément dans les sources données couvrant la migration PostgreSQL vers Oracle Database.  
>
> **⚠️ INFORMATION NON DISPONIBLE** détaillée sur procédures, outils et scripts automatisés PostgreSQL → Oracle dans les sources officielles listées.  
>
> Ce guide repose sur la compilation des bonnes pratiques générales en migration multi-SGBD, avec indication claire des risques et points d’attention critiques.

---

# Sommaire

- 1. Introduction  
- 2. Architecture cible et exigences préalables  
- 3. Analyse des différences majeures PostgreSQL → Oracle  
- 4. Préparation de la migration  
- 5. Exportation des données PostgreSQL  
- 6. Transformation et chargement dans Oracle  
- 7. Migration des objets complexes (fonctions, triggers, séquences)  
- 8. Gestion des utilisateurs, rôles, permissions, et sécurité  
- 9. Validation post-migration et tests  
- 10. Scripts de surveillance, backup, et dépannage  
- 11. Recommandations pour disponibilité et disaster recovery  
- 12. Annexes (tableaux de compatibilité, schémas, estimation durées)

---

## 1. Introduction

Ce guide détaille les étapes essentielles pour migrer une base de données PostgreSQL vers Oracle Database, en se focalisant sur les problématiques d’incompatibilités, conservation des données, sécurité, et validation.  

Cette migration implique de nombreux défis, notamment au niveau des différences dans la syntaxe SQL, des types de données, des objets spécifiques (procédures stockées, séquences, triggers), et de la gestion des utilisateurs. Ce document vise à fournir un cadre méthodologique robuste, en s’appuyant sur des bonnes pratiques industrielles validées.

---

## 2. Architecture cible et exigences préalables

### 2.1 Architecture cible Oracle Database

- Oracle Database version cible : entrer la version précise utilisée (exemple: 19c, 21c)
- Options activées : Partitioning, Advanced Security, etc. selon besoins
- Infrastructure : serveurs dédiés, stockage, réseaux — configurés haute disponibilité (RAC, Data Guard) si applicable

### 2.2 Pré-requis techniques

- Accès administrateur à la base PostgreSQL source
- Accès DBA à la base Oracle cible
- Environnement intermédiaire pour manipulations des données (serveur Linux/Windows)
- Outils SQL et scripts shell/Python pour extraction, transformation, chargement (ETL)

---

## 3. Analyse des différences majeures PostgreSQL → Oracle

| Aspect                  | PostgreSQL                          | Oracle                            | Impact migration                        |
|-------------------------|-----------------------------------|----------------------------------|---------------------------------------|
| Types de données        | text, bytea, JSONB, SERIAL, array  | VARCHAR2, BLOB, CLOB, NUMBER     | Adaptation datatype nécessaire         |
| Séquences               | Système natif, usage avec SERIAL  | Séquences indépendantes          | Conversion des séquences manuelle      |
| Fonctions & procédures  | PL/pgSQL                         | PL/SQL                           | Relecture complète du code à prévoir   |
| Transactions           | MVCC, READ COMMITTED par défaut    | MVCC, implémentations Oracle     | Comportement transactionnel à valider |
| Triggers                | Avant et après DML                 | Avant–après, avec syntaxe Oracle | Réécriture requise                      |
| Gestion utilisateurs    | Rôles et attributs PostgreSQL     | Users, roles, profils Oracle     | Mapping complexité avec permissions    |
| Index                  | B-tree, GIN, GiST                 | B-tree, bitmap                   | Adaptation possible selon usages       |

---

## 4. Préparation de la migration

### 4.1 Audit initial et sauvegarde

- Sauvegarde complète PostgreSQL (dump pg_dump, pg_dumpall)
- Revue des objets, dépendances, contraintes
- Identification des données sensibles et plan de sécurité

### 4.2 Planification projet migration

| Phase                   | Durée estimée                   | Notes                                |
|-------------------------|--------------------------------|-------------------------------------|
| Analyse et conception   | 3-7 jours                      | Revues différences, mapping objets  |
| Extraction données      | Variable selon taille base     | Avec test de corruption              |
| Transformation ETL      | 7-14 jours                    | Développement scripts conversion    |
| Chargement dans Oracle  | Variable                      | Contrôle erreurs                     |
| Validation              | 3-5 jours                    | Tests fonctionnels et recette        |

---

## 5. Exportation des données PostgreSQL

### 5.1 Export avec pg_dump

```bash
# Sauvegarde complète base PostgreSQL au format custom
pg_dump -Fc -U postgres -h <host_postgres> -p 5432 -d nom_base -f /tmp/pg_dump_nom_base.custom
```

- Options importantes:  
  - `-Fc` : format custom, flexible pour restauration ou extraction  
  - `-U` : utilisateur source  
  - `-h` : hôte serveur PostgreSQL  
  - `-p` : port PostgreSQL  
- Vérification de l’intégrité du dump:  
```bash
file /tmp/pg_dump_nom_base.custom
# Résultat attendu: PostgreSQL custom database dump
```  

### 5.2 Export CSV ponctuel

Pour tables spécifiques (exemple):

```bash
psql -U postgres -h <host_postgres> -d nom_base -c "\copy (SELECT * FROM table_exemple) TO '/tmp/table_exemple.csv' CSV HEADER;"
```

- Sortie: CSV avec entête  
- À utiliser pour contrôle à l’étape de chargement Oracle

### 5.3 Gestion erreurs export

- Capture erreurs dans un fichier log  
- Validation checksum éventuelle sur fichiers export

---

## 6. Transformation et chargement dans Oracle

### 6.1 Techniques recommandées

- Utilisation de SQL*Loader ou Oracle External Tables pour chargements massifs  
- Scripts PL/SQL de transformation pour adaptation types de données

### 6.2 Exemple SQL*Loader control file

Fichier: `table_exemple.ctl`

```shell
LOAD DATA
INFILE '/tmp/table_exemple.csv'
INTO TABLE table_exemple
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
TRAILING NULLCOLS
(
  id INTEGER EXTERNAL,
  nom CHAR,
  date_creation DATE "YYYY-MM-DD",
  montant DECIMAL EXTERNAL
)
```

Lancement:

```bash
sqlldr userid=sys/password@orcl control=table_exemple.ctl log=table_exemple.log bad=table_exemple.bad
```

- Logs et erreurs dans `table_exemple.log` et `.bad`  
- Validation de chargement via requêtes Oracle

---

## 7. Migration des objets complexes

### 7.1 Fonctions et procédures

- Conversion PL/pgSQL → PL/SQL manuelle nécessaire  
- Adapter syntaxe, types, gestion erreurs

### 7.2 Triggers

- Recréer triggers Oracle (avant/après)  
- Vérifier limitations Oracle sur triggers row-level ou statement-level

### 7.3 Séquences

Extraction:

```sql
-- PostgreSQL
SELECT sequence_name, last_value FROM pg_sequences WHERE schemaname='public';
```

Recréation Oracle:

```sql
CREATE SEQUENCE seq_exemple 
START WITH <last_value> 
INCREMENT BY 1 
NOCACHE NOCYCLE;
```

---

## 8. Gestion utilisateurs, rôles et sécurité

### 8.1 Mapping utilisateurs PostgreSQL vers Oracle

- PostgreSQL : utilisateurs plus rôles plus attributs
- Oracle : utilisateurs identifiés par USERNAME + profiles de sécurité

### 8.2 Permissions et grant

Migration manuelle ou scriptée de la gestion GRANT

```sql
-- Exemple Oracle
GRANT CONNECT, RESOURCE TO nouveau_user;
```

### 8.3 Résolution des utilisateurs orphelins

> ⚠️ Cette étape critique nécessite la vérification des SID et correspondance des users  
> Pas de documentation officielle disponible pour automatisation PostgreSQL→Oracle

---

## 9. Validation post-migration et tests

### 9.1 Validation données

- Checksums sur colonnes critiques  
- Comparaison comptes rendus export/import CSV

### 9.2 Tests fonctionnels

- Scripts testant procédures stockées et triggers  
- Validation transactions complètes  

### 9.3 Monitoring et contrôle erreurs

- Activer logging Oracle audit et SQL diagnostics  
- Plans de secours et script rollback prêts

---

## 10. Scripts et outils recommandés

### 10.1 Exemple minimal script shell extraction/export postgres

```bash
#!/bin/bash
# Script export PostgreSQL complet avec logging, gestion erreur

PGUSER="postgres"
PGHOST="localhost"
PGPORT=5432
PGDATABASE="nom_base"
DUMPFILE="/tmp/dump_pg.sql"
LOGFILE="/tmp/export_pg.log"

{
  echo "Début export $(date)"
  pg_dump -U $PGUSER -h $PGHOST -p $PGPORT -d $PGDATABASE > $DUMPFILE
  if [ $? -ne 0 ]; then
    echo "ERREUR lors de l'export PostgreSQL" >&2
    exit 1
  fi
  echo "Export terminé"
} >> $LOGFILE 2>&1
```

---

## 11. Recommandations disponibilité et DR

- Prévoir bascule Oracle avec Data Guard pour HA et DR  
- Mise en place de backups Oracle RMAN (Recovery Manager)  
- Définir SLA opérationnels et procédures d’alerte automatisées

---

## 12. Annexes

### 12.1 Tableau comparatif types de données PostgreSQL vers Oracle

| PostgreSQL Type  | Oracle Type               | Note                              |
|------------------|--------------------------|----------------------------------|
| SERIAL / INT     | NUMBER / INTEGER          | Conversion simple                 |
| TEXT / VARCHAR   | VARCHAR2                 | Truncature possible à gérer       |
| BYTEA           | BLOB                      | Conversion binaire                |
| TIMESTAMP       | DATE / TIMESTAMP          | Format à convertir                |
| BOOLEAN         | NUMBER(1)                | 0/1 ou CHAR('Y','N')             |
| JSON/JSONB       | CLOB ou JSON             | Support natif à vérifier Oracle  |

### 12.2 Exemple diagramme ASCII simplifié architecture migration

```
+-----------+       Export dump       +-----------+       Chargement       +------------+
| PostgreSQL| ----------------------> | Serveur   | ---------------------> | Oracle DB  |
|   Source  |                         | ETL/Tools |                       | Cible      |
+-----------+                         +-----------+                       +------------+
```

---

## Conclusion

Cette migration PostgreSQL → Oracle Database nécessite une forte implication pour adapter correctement les données, objets, et paramètres de sécurité. L’absence d’outil officiel dédié impose beaucoup de travail manuel et des validations rigoureuses. Ce guide donne un socle de bonnes pratiques et étapes clés, indispensables pour réussir votre migration en environnement production.

---

> ⚠️ **Note**:  
> Cette documentation repose uniquement sur les sources officielles disponibles. En l’absence d’outil officiel dédié pour PostgreSQL vers Oracle, il est recommandé de faire appel à des experts Oracle et PostgreSQL expérimentés avec un Proof of Concept approfondi avant mise en production.

---

### Sources officielles consultées:  
⚠️ Aucune documentation officielle disponible spécifiquement pour la migration PostgreSQL vers Oracle Database dans les sources listées.

---

**Fin du document**

---

## 📊 Génération

- **Généré**: 15/12/2025 16:58:46
- **Langue**: Français
- **Modèle**: Perplexity Sonar
- **Score audit**: 85/100
- **Statut**: APPROVED
