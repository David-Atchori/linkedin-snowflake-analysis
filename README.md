# Projet LinkedIn — Analyse des Offres d'Emploi avec Snowflake

## Description

Ce projet analyse plusieurs milliers d'offres d'emploi LinkedIn en utilisant Snowflake comme entrepôt de données et Streamlit pour les visualisations. Il suit une architecture Medallion (Bronze / Silver / Gold) pour garantir la qualité et la traçabilité des données.

---

## Architecture des données

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
│ company_name (FK) ──┼────────│ name             │
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

## Structure du projet

Tout le code est contenu dans un seul notebook `LinkedIn_Lab_Final.ipynb`, organisé en 12 étapes séquentielles :

| Étape | Contenu |
|-------|---------|
| 1 | Création de la base de données `linkedin` et des 3 schémas (Bronze, Silver, Gold) |
| 2 | Création du stage S3 externe et des formats de fichiers (CSV et JSON) |
| 3 | Création des tables Bronze (tout en VARCHAR / VARIANT) |
| 4 | Chargement des données depuis S3 via `COPY INTO` |
| 5 | Analyse exploratoire des données Bronze (valeurs nulles, valeurs distinctes, structure JSON) |
| 6 | Création des tables Silver avec les bons types |
| 7 | Transformation Bronze → Silver (typage, déduplification, nettoyage) |
| 8 | Tests de qualité des données Silver (doublons, nulls, cohérence, clés orphelines) |
| 9 | Création des tables Gold (données pré-agrégées pour chaque analyse) |
| 10 | Requêtes SQL des 5 analyses |
| 11 | Automatisation avec Snowpipe et tâches planifiées |
| 12 | Code Streamlit des 5 visualisations |

---

## Jeu de données

Les fichiers sont disponibles dans le bucket S3 public : `s3://snowflake-lab-bucket/`

| Fichier | Format | Description |
|---------|--------|-------------|
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

## Instructions pour reproduire le projet

1. Créer un compte Snowflake
2. Ouvrir le notebook `LinkedIn_Lab_Final.ipynb`
3. Exécuter les cellules dans l'ordre (étapes 1 à 11)
4. Pour Streamlit : créer une Streamlit App dans Snowflake (Run on warehouse), coller le code de l'étape 12 et cliquer sur Run

---

## Analyses réalisées

### Analyse 1 — Top 10 des titres de postes les plus publiés par industrie

Jointure entre `job_postings` et `company_industries` via `company_id`. `ROW_NUMBER() OVER (PARTITION BY industry)` + `QUALIFY` pour garder le top 10 par secteur. L'app Streamlit propose un menu déroulant pour filtrer par industrie.

![Analyse 1](Analyse%201%20-%20Top%2010%20des%20titres%20de%20postes%20les%20plus%20publi%C3%A9s%20par%20industrie.png)

### Analyse 2 — Top 10 des postes les mieux rémunérés par industrie

`AVG(max_salary)` groupé par industrie et titre, avec filtre `WHERE max_salary IS NOT NULL`. Même logique de `QUALIFY` pour le top 10 par secteur.

![Analyse 2](Analyse%202%20-%20Top%2010%20des%20postes%20les%20mieux%20r%C3%A9mun%C3%A9r%C3%A9s%20par%20industrie.png)

### Analyse 3 — Répartition des offres par taille d'entreprise

`CASE WHEN company_size` traduit les valeurs 0–7 en labels lisibles ("Très petite" à "Géante"). Jointure `job_postings → companies`.

![Analyse 3](Analyse%203%20-%20R%C3%A9partition%20des%20offres%20par%20taille%20d'entreprise.png)

### Analyse 4 — Répartition des offres par secteur d'activité

Top 20 des secteurs (`LIMIT 20`) avec `COUNT(*)` groupé sur `company_industries.industry`.

![Analyse 4](Analyse%204%20-%20R%C3%A9partition%20des%20offres%20par%20secteur%20d'activit%C3%A9.png)

### Analyse 5 — Répartition des offres par type d'emploi

`COUNT(*)` groupé sur `formatted_work_type` (Full-time, Part-time, Contract, Internship, etc.), avec filtre `IS NOT NULL`.

![Analyse 5](Analyse%205%20-%20R%C3%A9partition%20des%20offres%20par%20type%20d'emploi.png)

---

## Automatisation

Le projet implémente un pipeline automatisé complet :

- **Snowpipe** (`AUTO_INGEST = TRUE`) — un pipe par fichier source, charge automatiquement les nouveaux fichiers S3 dans Bronze dès leur arrivée. Note : l'`AUTO_INGEST` nécessite une configuration AWS (S3 → SQS) non disponible sur un bucket public ; les pipes sont créés à titre de démonstration.
- **Tâche `refresh_silver`** — s'exécute toutes les 60 minutes, rejoue les transformations Bronze → Silver avec `TRUNCATE + INSERT`.
- **Tâche `refresh_gold`** — s'exécute toutes les 70 minutes (décalée de 10 min pour s'exécuter après Silver), recalcule toutes les tables Gold.

---

## Problèmes rencontrés et solutions

| Problème | Solution |
|----------|----------|
| Timestamps au format Unix millisecondes | `TO_TIMESTAMP_NTZ(CAST(TRY_TO_DOUBLE(valeur) / 1000 AS BIGINT))` |
| `company_name` contient des IDs flottants (ex : `7789.0`) | `SPLIT_PART(company_name, '.', 1)` pour supprimer le `.0` |
| Doublons dans `job_postings` | `ROW_NUMBER() OVER (PARTITION BY job_id) + WHERE rn = 1` |
| Fichiers JSON avec structure imbriquée | Stockage en `VARIANT` dans Bronze, extraction avec `raw_data:champ::TYPE` dans Silver |
| `state` et `zip_code` avec valeur `'0'` (données incorrectes) | `NULLIF(raw_data:state::VARCHAR, '0')` |
| `time_recorded` dans `employee_counts` en secondes (pas millisecondes) | Pas de division par 1000 pour cette colonne |

---

## Répartition des tâches

| Membre | Tâches |
|--------|--------|
| David ATCHORI | Étapes 1–8 : setup, Bronze, Silver, tests qualité |
| Cédric Don-Agoh BOIMIN | Gold, automatisation  |
| Eya ben REJEB |  Streamlit, README  |
