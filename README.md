# Projet LinkedIn — Analyse des Offres d'Emploi avec Snowflake

## Description

Ce projet analyse plusieurs milliers d'offres d'emploi LinkedIn en utilisant Snowflake comme entrepôt de données et Streamlit pour les visualisations. Il met en oeuvre une architecture Medallion (Bronze / Silver / Gold) pour garantir la qualité et la traçabilité des données.

---

## Architecture des Données

```
S3 (Fichiers bruts)
       ↓
   BRONZE        → Données brutes telles quelles (tout en VARCHAR / VARIANT)
       ↓
   SILVER        → Données nettoyées, typées, dédupliquées
       ↓
   GOLD          → Données agrégées prêtes pour l'analyse
       ↓
   STREAMLIT     → Visualisations interactives
```

---

## Diagramme ERD

```
BRONZE / SILVER
┌─────────────────────┐        ┌──────────────────┐
│    job_postings     │        │    companies     │
│─────────────────────│        │──────────────────│
│ job_id (PK)         │        │ company_id (PK)  │
│ company_name (FK)───┼────────│ name             │
│ title               │        │ description      │
│ description         │        │ company_size     │
│ max_salary          │        │ state            │
│ med_salary          │        │ country          │
│ min_salary          │        │ city             │
│ pay_period          │        │ zip_code         │
│ formatted_work_type │        │ address          │
│ location            │        │ url              │
│ applies             │        └──────────────────┘
│ original_listed_time│              │
│ remote_allowed      │              │
│ views               │        ┌─────┴────────────┐
│ listed_time         │        │company_industries│
│ expiry              │        │──────────────────│
│ sponsored           │        │ company_id (FK)  │
│ currency            │        │ industry         │
│ compensation_type   │        └──────────────────┘
└─────────────────────┘
         │                     ┌──────────────────────┐
         │                     │ company_specialities │
    ┌────┴──────┐               │──────────────────────│
    │  benefits │               │ company_id (FK)      │
    │───────────│               │ speciality           │
    │ job_id(FK)│               └──────────────────────┘
    │ inferred  │
    │ type      │         ┌──────────────────┐
    └───────────┘         │  job_industries  │
                          │──────────────────│
    ┌───────────┐         │ job_id (FK)      │
    │ job_skills│         │ industry_id      │
    │───────────│         └──────────────────┘
    │ job_id(FK)│
    │ skill_abr │         ┌──────────────────────┐
    └───────────┘         │  employee_counts     │
                          │──────────────────────│
                          │ company_id (FK)      │
                          │ employee_count       │
                          │ follower_count       │
                          │ time_recorded        │
                          └──────────────────────┘

GOLD
┌──────────────────────────────┐
│  top_titres_par_industrie    │
│  top_salaires_par_industrie  │
│  repartition_taille_entreprise│
│  repartition_secteur         │
│  repartition_type_emploi     │
└──────────────────────────────┘
```

---

## Structure du Projet

```
repo-linkedin-snowflake/
│
├── README.md
│
├── sql/
│   ├── 01_setup.sql              → Création BDD, schémas, stage, formats
│   ├── 02_tables_bronze.sql      → Création des tables Bronze
│   ├── 03_load_bronze.sql        → Chargement des données dans Bronze
│   ├── 04_tables_silver.sql      → Création des tables Silver
│   ├── 05_load_silver.sql        → Transformation Bronze → Silver
│   ├── 06_quality_tests.sql      → Tests de qualité des données
│   ├── 07_tables_gold.sql        → Création et alimentation des tables Gold
│   ├── 08_analyses.sql           → Requêtes SQL des 5 analyses
│   └── 09_automation.sql         → Snowpipe + tâches planifiées
│
└── streamlit/
    └── streamlit_app.py          → Code Streamlit des 5 visualisations
```

---

## Jeu de Données

Les fichiers sont disponibles dans le bucket S3 : `s3://snowflake-lab-bucket/`

| Fichier | Format | Description |
|---|---|---|
| `job_postings.csv` | CSV | Offres d'emploi détaillées |
| `benefits.csv` | CSV | Avantages par offre |
| `employee_counts.csv` | CSV | Nombre d'employés par entreprise |
| `job_skills.csv` | CSV | Compétences par offre |
| `companies.json` | JSON | Informations sur les entreprises |
| `company_industries.json` | JSON | Secteurs d'activité par entreprise |
| `company_specialities.json` | JSON | Spécialités par entreprise |
| `job_industries.json` | JSON | Secteurs d'activité par offre |

---

## Prérequis

- Compte Snowflake (gratuit 120 jours)
- Connaissances de base en SQL et Python

---

## Instructions pour Reproduire le Projet

1. Créer un compte Snowflake
2. Exécuter les scripts SQL dans l'ordre (01 à 09)
3. Créer une Streamlit app dans Snowflake (Run on warehouse)
4. Coller le code `streamlit_app.py` et cliquer sur Run

---

## Analyses Réalisées

### Analyse 1 — Top 10 des titres de postes les plus publiés par industrie
Identifie les titres de postes les plus demandés pour chaque secteur d'activité. Permet de repérer les métiers en tension par industrie.

### Analyse 2 — Top 10 des postes les mieux rémunérés par industrie
Compare les salaires maximaux moyens par poste et par industrie. Utile pour identifier les opportunités de carrière les mieux rémunérées.

### Analyse 3 — Répartition des offres par taille d'entreprise
Montre si les offres d'emploi proviennent majoritairement de grandes ou petites entreprises. La taille va de 0 (très petite) à 7 (géante).

### Analyse 4 — Répartition des offres par secteur d'activité
Identifie les 20 secteurs qui recrutent le plus sur LinkedIn. Donne une vue d'ensemble du marché de l'emploi par industrie.

### Analyse 5 — Répartition des offres par type d'emploi
Compare le nombre d'offres selon le type de contrat : temps plein, temps partiel, stage, contrat, etc.

---

## Automatisation

Le projet utilise :
- **Snowpipe** (`AUTO_INGEST = TRUE`) pour charger automatiquement les nouveaux fichiers S3 dans Bronze
- **Tâches planifiées** pour propager les données de Bronze vers Silver (toutes les 60 minutes) puis vers Gold (toutes les 70 minutes)

---

## Problèmes Rencontrés et Solutions

| Problème | Solution |
|---|---|
| Timestamps au format Unix millisecondes | Conversion avec `TO_TIMESTAMP_NTZ(CAST(valeur/1000 AS BIGINT))` |
| `company_name` contient des IDs flottants (ex: `7789.0`) | Nettoyage avec `SPLIT_PART(company_name, '.', 1)` |
| Doublons dans `job_postings` | Déduplification avec `ROW_NUMBER() + QUALIFY` |
| Fichiers JSON avec structure imbriquée | Stockage en `VARIANT` dans Bronze, extraction dans Silver |

---

## Répartition des Tâches

| Membre | Tâches |
|---|---|
| Membre 1 | Setup, Bronze, Silver, Tests qualité |
| Membre 2 | Gold, Streamlit, Automatisation, README |

## Résultats des visualisations

### Analyse 1 - Top 10 des titres de postes les plus publiés par industrie
![Résultat Analyse 1](Top 10 des titres de postes les plus publiés par industrie.png)

### Analyse 2 - Top 10 des postes les mieux rémunérés par industrie
![Résultat Analyse 2](Top 10 des postes les mieux rémunérés par industrie.png)

### Analyse 3 - Répartition des offres par taille d'entreprise
![Résultat Analyse 3](Répartition des offres par taille d'entreprise.png)

### Analyse 4 - Répartition des offres par secteur d'activité
![Résultat Analyse 4](Répartition des offres par secteur d'activité.png)

### Analyse 5 - Répartition des offres par type d'emploi
![Résultat Analyse 5](Analyse 5 - Répartition des offres par type d'emploi.png)
