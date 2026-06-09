# Soluna — Plateforme de données

Plateforme de données pour **Soluna**, marque (fictive) de compléments alimentaires
personnalisés par abonnement. Le dépôt regroupe l'infrastructure (Infrastructure as Code),
la modélisation des données (Bloc 2) et un pipeline de données temps réel (Bloc 3).

> **Note sur les données.** Le jeu **Instacart Market Basket Analysis** (Kaggle) sert de
> *proxy réel* à l'historique de commandes et à la cadence de réachat de Soluna. Soluna est
> une entité fictive ; les données de quiz de bien-être évoquées dans la gouvernance sont synthétiques.

## Architecture

Deux flux complémentaires, tout conteneurisé (Docker) et décrit en Infrastructure as Code :

- **Batch — entrepôt analytique.** Sources → chargement SQL → entrepôt **PostgreSQL** en deux couches :
  - `raw` : copie fidèle des fichiers sources (6 tables).
  - `mart` : schéma en étoile (Kimball) — 1 table de faits + 3 dimensions.
- **Temps réel — pipeline de cadence.** Les commandes sont publiées dans **Kafka**
  (producteur), consommées et transformées en continu (consommateur) puis chargées dans la
  couche `stream`. **Airflow** orchestre des contrôles qualité et le rafraîchissement d'un
  agrégat de segmentation (fréquent / régulier / à risque).

## Stack technique

| Composant | Rôle |
|-----------|------|
| PostgreSQL 16 | Entrepôt de données (couches `raw`, `mart`, `stream`) |
| Docker / docker-compose | Conteneurs + Infrastructure as Code |
| Apache Kafka | Bus d'événements temps réel |
| Apache Airflow | Orchestration et planification |
| Python (confluent-kafka, psycopg2) | Producteur / consommateur du flux |
| Adminer | Exploration de la base (http://localhost:8080) |

## Prérequis

- Docker Desktop
- Python 3 avec `confluent-kafka` et `psycopg2-binary` (`pip install confluent-kafka psycopg2-binary`)
- Les fichiers CSV Instacart placés dans `data/` (non versionnés)

## Mise en route

### 1. Démarrer l'infrastructure
```bash
docker compose up -d
```

### 2. Construire l'entrepôt (batch)
```bash
docker compose exec -T warehouse psql -U soluna -d soluna < sql/init.sql       # couche raw
docker compose exec -T warehouse psql -U soluna -d soluna < sql/02_star.sql    # couche mart (etoile)
docker compose exec -T warehouse psql -U soluna -d soluna < sql/03_stream.sql  # couche stream
```

### 3. Lancer le pipeline temps réel
```bash
# Creer le canal d'evenements (une fois)
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders --partitions 3 --replication-factor 1

# Terminal A : publier les commandes dans Kafka
python python/producer.py

# Terminal B : consommer, valider, transformer et charger les indicateurs
python python/consumer.py
```

### 4. Orchestration (Airflow)
Interface : http://localhost:8081 (utilisateur `admin`, mot de passe affiché par
`docker compose exec airflow cat /opt/airflow/standalone_admin_password.txt`).
Activer puis déclencher le DAG **`soluna_pipeline`** : il contrôle la qualité du flux et
rafraîchit la table `stream.cadence_resume` (segmentation des abonnés).

## Modèle de données

**Couche `mart` (analytique).**
- `fait_commande_produit` — grain : 1 produit acheté dans une commande (~34 M lignes).
- `dim_client`, `dim_produit`, `dim_temps`.

**Couche `stream` (temps réel).**
- `client_features` — indicateurs par abonné mis à jour en continu (nombre de commandes, cadence).
- `client_cadence` (vue) — cadence moyenne lisible par abonné.
- `evenements_invalides` — file d'erreurs (événements rejetés par le contrôle qualité).
- `cadence_resume` — segmentation des abonnés produite par Airflow.

## Structure du dépôt

```
soluna/
├── docker-compose.yml      # Infrastructure as Code (PostgreSQL, Adminer, Kafka, Airflow)
├── sql/
│   ├── init.sql            # couche raw : creation + chargement
│   ├── 02_star.sql         # couche mart : schema en etoile
│   └── 03_stream.sql       # couche stream : tables temps reel
├── python/
│   ├── producer.py         # publie les commandes dans Kafka
│   └── consumer.py         # consomme, valide, transforme, charge
├── dags/
│   └── soluna_pipeline.py  # DAG Airflow : controle qualite + agregat
├── data/                   # CSV Instacart (non versionnes)
└── README.md
```

## Suite du projet

- **Bloc 4** — modèle d'IA anti-churn (à partir de la couche `stream`), API de serving,
  CI/CD et monitoring en production.
