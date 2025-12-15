# Migration Oracle 19c vers Azure PostgreSQL Flexible Server 500GB

---

> **📋**: Guide expert pour effectuer la migration d’une base de données Oracle 19c vers Azure PostgreSQL Flexible Server avec un volume de données de 500GB.  
> **🏷️**: Migration base de données, Oracle, PostgreSQL, Azure, Cloud  
> **🌍**: Français  
> **📊**: Hybride  
> **📅**: 27/04/2024  

## 📋 Table

- Vue d'ensemble  
- Architecture  
- Prérequis  
- Installation  
- Configuration  
- Migration  
- Sécurité  
- Déploiement  
- Monitoring  
- Troubleshooting  
- Ressources  

---

## Vue d'ensemble

La migration d'Oracle 19c vers Azure PostgreSQL Flexible Server est une opération stratégique pour moderniser vos bases de données en tirant parti des services cloud managés d’Azure. Cette migration concerne ici une base de données d’environ 500 GB, ce qui nécessite une planification rigoureuse afin de minimiser les interruptions, garantir l'intégrité des données, et optimiser la performance post-migration.

Azure Database for PostgreSQL Flexible Server offre une architecture cloud scalable, résiliente, et compatible avec les standards PostgreSQL, tout en intégrant des fonctionnalités managées adaptées aux charges d'entreprise.

---

## Architecture

- **Source** : Oracle Database 19c on-premise ou IaaS  
- **Cible** : Azure PostgreSQL Flexible Server (Tier General Purpose recommandé pour 500GB)  
- **Connectivité** : VPN ou ExpressRoute pour une migration sécurisée et performante  
- **Composants intermédiaires** : outils de migration (Azure DMS, ora2pg, Data Factory)  
- **Sécurité** : Gestion des identités Azure AD, chiffrement au repos et en transit  

---

## Prérequis

- **Accès administrateur** sur la base Oracle 19c source  
- **Azure Subscription** avec droits pour provisionner PostgreSQL Flexible Server  
- Taille disponible sur le serveur cible > 500GB + espace pour les opérations internes  
- Outils suivants installés :  
  - Azure Data Migration Service (DMS)  
  - ora2pg (outil open-source de migration Oracle->PostgreSQL)  
  - Azure CLI  
- Connexion réseau sécurisée entre source et Azure (VPN ou ExpressRoute)  
- Compétences en SQL Oracle et PostgreSQL, ainsi que connaissance approfondie de la gestion de bases cloud  

---

## Installation

- Installer et configurer Azure CLI (version récente)  
- Provisionner un serveur Azure PostgreSQL Flexible Server via Azure Portal ou CLI avec :  
  - Version PostgreSQL compatible (exemple : 13 ou 14)  
  - Tier General Purpose avec stockage ≥ 512GB  
  - Configuration haute disponibilité si besoin (mode zone-redundant disponible)  
- Installer ora2pg sur un serveur intermédiaire (Linux recommandé)  
- Installer et configurer Azure DMS si migration en ligne est envisagée  

---

## Configuration

### Configuration PostgreSQL Flexible Server

- Activer le paramètre `max_connections` selon la charge estimée  
- Paramétrer les tablespaces si possible pour optimiser I/O  
- Vérifier et ajuster les paramètres `work_mem`, `maintenance_work_mem` en fonction de la mémoire  
- Configurer les sauvegardes automatiques et RPO souhaité  
- Configurer le firewall Azure pour autoriser uniquement les IP/services nécessaires  

### Configuration Oracle source

- Activer l’archive log si mode recovery point-in-time requis  
- Créer et accorder les rôles nécessaires/privileges pour accéder aux données  
- Préparer pour export dans un format compatible (expdp/impdp selon besoin)  

---

## Migration

### Étape 1 : Analyse et cartographie des données

- Utiliser ora2pg pour extraire un rapport d’évaluation du schéma Oracle et des types incompatibles  
- Adapter le schéma PostgreSQL par conversion (types, contraintes, procédures, triggers)  
- Documenter les changements manuels nécessaires (PL/SQL vs PL/pgSQL différences)  

### Étape 2 : Extraction du schéma

- Avec ora2pg générer les scripts DDL PostgreSQL corrigés  
- Déployer le schéma sur Azure PostgreSQL Flexible Server  

### Étape 3 : Migration des données

- Exporter les données via Data Pump ou via un export CSV/Parquet selon performance et downtime  
- Charger dans PostgreSQL avec `COPY` ou via Azure Data Factory  
- Valider l’intégrité des données par échantillonnage ou checksum  

### Étape 4 : Migration des procédures stockées et jobs

- Traduire manuellement ou avec outil les PL/SQL vers PL/pgSQL  
- Redéployer les jobs avec des outils tiers ou via pgAgent  

### Étape 5 : Validation finale

- Contrôler la cohérence des données  
- Effectuer des tests fonctionnels appliqués aux changements PostgreSQL  
- Simuler bascule et valider les performances  

---

## Sécurité

- Utiliser Azure AD pour l’authentification PostgreSQL si possible  
- Configurer le chiffrement TLS pour les connexions client-serveur  
- Définir un RBAC strict autour des comptes et ressources Azure  
- Activer le logging d’audit PostgreSQL et centraliser via Azure Monitor  
- Sauvegarder avec Azure Backup et définir une stratégie de conservation  

---

## Déploiement

- Planifier la bascule en heures creuses avec procédure rollback claire  
- Mettre hors ligne Oracle ou configurer un mode read-only si bascule finale longue  
- Piloter la bascule DNS ou modification des endpoints applicatifs vers Azure PostgreSQL  
- Surveiller étroitement après bascule pour corriger les anomalies rapidement  

---

## Monitoring

- Utiliser Azure Monitor + PostgreSQL Insights pour métriques et alertes  
- Analyser les requêtes lentes avec pg_stat_statements  
- Superviser les consommations CPU, mémoire, IOPS et latences  
- Mettre en place des tableaux de bord personnalisés dans Azure Portal  

---

## Troubleshooting

- Vérifier les incompatibilités de type ou fonction lors des imports de données  
- Analyser les erreurs de connexion réseau (firewall, VPN)  
- Contrôler l’espace disque et quotas du serveur Flexible Server  
- Examiner les logs PostgreSQL pour erreurs ou timeout  
- Ajuster les paramètres de configuration PostgreSQL en cas de contention  

---

## Ressources

1. https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/  
2. https://oracle-base.com/articles/19c/whats-new-in-oracle-database-19c  
3. https://ora2pg.darold.net/  
4. https://learn.microsoft.com/en-us/azure/dms/tutorial-migrate-oracle-to-azure-postgresql  
5. https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-secure-connections  

---

## 📝 Changelog

### v1.0 (2024-04-27)  
- Création  
- 5 sources principales  
- Hybride (documentation officielle et usages terrain)

---

## 📊 Génération

- **Généré**: 15/12/2025 14:15:17
- **Langue**: Français
- **Modèle**: Perplexity Sonar
- **Score audit**: 85/100
- **Statut**: APPROVED
