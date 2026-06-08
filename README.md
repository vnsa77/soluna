# Soluna — Plateforme de données

Entrepôt de données et infrastructure pour **Soluna**, marque (fictive) de compléments
alimentaires personnalisés par abonnement. Ce dépôt regroupe l'infrastructure décrite en
*Infrastructure as Code* et la modélisation des données (projet de mémoire — Bloc 2 : Architecture).

> **Note sur les données.** Le jeu **Instacart Market Basket Analysis** (Kaggle) sert de
> *proxy réel* à l'historique de commandes et à la cadence de réachat de Soluna. Soluna est
> une entité fictive ; les données de quiz de bien-être évoquées dans la gouvernance sont synthétiques.

## Architecture

Sources → ingestion (chargement SQL) → entrepôt **PostgreSQL** conteneurisé → consommation
(analyse SQL / BI, et modèle d'IA anti-churn dans un bloc ultérieur). L'entrepôt est organisé en deux couches :

- **`raw`** — copie fidèle des fichiers sources (6 tables), sans transformation.
- **`mart`** — schéma en étoile (modèle de Kimball) : 1 table de faits + 3 dimensions.

Toute l'infrastructure est déclarée dans `docker-compose.yml` et déployée en une seule commande.

## Stack technique

| Composant | Rôle |
|-----------|------|
| PostgreSQL 16 | Entrepôt de données |
| Docker / docker-compose | Conteneurs + Infrastructure as Code |
| Adminer | Exploration de la base (http://localhost:8080) |

## Prérequis

- Docker Desktop
- Les fichiers CSV Instacart placés dans le dossier `data/` (non versionnés)

## Démarrage

```bash
# 1. Lancer l'entrepôt + l'outil d'exploration
docker compose up -d

# 2. Charger les données sources (couche raw)
docker compose exec -T warehouse psql -U soluna -d soluna < sql/init.sql

# 3. Construire le schéma en étoile (couche mart)
docker compose exec -T warehouse psql -U soluna -d soluna < sql/02_star.sql
```

Adminer : http://localhost:8080 — Système *PostgreSQL*, Serveur `warehouse`,
utilisateur `soluna`, mot de passe `soluna`, base `soluna`.

## Modèle de données (couche `mart`)

- `fait_commande_produit` — grain : 1 produit acheté dans une commande (~34 M lignes).
- `dim_client` — abonné + cadence moyenne entre commandes (signal clé de l'anti-churn).
- `dim_produit` — produit, rayon, département.
- `dim_temps` — jour, heure, moment de la journée.

## Structure du dépôt

```
soluna/
├── docker-compose.yml   # Infrastructure as Code (entrepôt + Adminer)
├── sql/
│   ├── init.sql         # couche raw : création des tables + chargement
│   └── 02_star.sql      # couche mart : schéma en étoile
├── data/                # CSV Instacart (non versionnés, voir .gitignore)
└── README.md
```

## Suite du projet

- **Bloc 3** — pipeline temps réel (Kafka) pour calculer la cadence de consommation en flux.
- **Bloc 4** — modèle d'IA anti-churn, API de serving, CI/CD et monitoring en production.
