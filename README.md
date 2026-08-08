# bixi-mobility-databricks-pipeline
🚲 BIXI Montréal — Pipeline de données Bronze/Silver/Gold sur Databricks

Pipeline de données end-to-end construit sur Databricks, appliquant l'architecture medallion (Bronze → Silver → Gold) aux données open data du réseau de vélos en libre-service BIXI Montréal.
Le dataset BIXI Montréal a été choisi pour sa simplicité : open data accessible, format CSV propre, et un volume confortable (13M+ trajets) suffisant pour illustrer un pipeline medallion réaliste sans complexité métier superflue.

Architecture

[Architecture du pipeline](screenshots/architecture.png)


N.B: L'ensemble du pipeline (Bronze → Silver → Gold) est orchestré via un Databricks Workflow planifié quotidiennement.

Stack technique:

1- Composant	Technologie
- Ingestion:	Auto Loader (cloudFiles), schéma évolutif
- Stockage:	Delta Lake
-Transformation:	PySpark
- Gouvernance	Unity Catalog (catalogs, schemas, volumes)
- Orchestration:	Databricks Workflows
- Restitution:	Databricks SQL Dashboard
- Langage; Python, SQL

2- Ce que fait chaque couche : 

-- Bronze — bronze.trips_raw Ingestion incrémentale des fichiers CSV bruts via Auto Loader, avec détection automatique du schéma et traçabilité (horodatage d'ingestion, fichier source via _metadata.file_path).

-- Silver — silver.trips_clean Nettoyage et mise en qualité : typage des colonnes temporelles (conversion epoch → timestamp, fuseau horaire America/Montreal), filtrage des trajets aberrants (durée < 1 min ou > 24h), suppression des lignes sans station valide, renommage en snake_case.

-- Gold — gold.daily_trips, gold.station_popularity, gold.seasonality

Trois agrégats orientés métier :

- Évolution du volume de trajets par jour
- Classement des stations les plus fréquentées (départs + arrivées)
- Répartition des trajets par jour de semaine et heure de la journée (saisonnalité)

Pourquoi Databricks SQL plutôt que Power BI ?

Le dashboard final a été construit directement dans Databricks SQL, plutôt que dans Power BI que je maîtrise déjà. Deux raisons à ce choix :
- Rapidité : le dashboard se construit directement sur les tables Gold, sans étape d'export ni de connecteur, ce qui boucle le projet entièrement dans l'environnement Databricks.
- Élargir mes méthodes de travail : sachant déjà faire des dashboards sur Power BI, je voulais apprendre à en construire aussi nativement sur Databricks, pour ajouter cet outil à ma boîte à outils plutôt que de me reposer uniquement sur ce que je connaissais déjà.
Une connexion Power BI reste possible en complément (voir schéma d'architecture), mais n'était pas l'objectif principal de cette étape.

Résultats clés

- 13M+ trajets bruts ingérés, ~12M après filtrage qualité
- Pics d'usage identifiés entre 17h et 18h en semaine, cohérents avec des trajets domicile-travail
- Stations du centre-ville et du campus universitaire largement en tête du classement

Défis rencontrés & résolutions

Quelques problèmes rencontrés en cours de route, résolus et documentés comme retour d'expérience :
- input_file_name() incompatible avec Unity Catalog → remplacé par la colonne native _metadata.file_path
- Décalage de fuseau horaire (+4h) entre les timestamps epoch (UTC) et l'heure réelle de Montréal → correction via from_utc_timestamp
- Checkpoint de streaming corrompu après plusieurs itérations de test → nettoyage propre (dbutils.fs.rm) plutôt qu'un contournement de chemin
- Un seul notebook pour 3 couches → refactoring en 3 notebooks distincts (un par couche), condition nécessaire pour une orchestration fiable via Databricks Workflows
  
Comment reproduire

- Télécharger les données trip history 2024 depuis bixi.com/en/open-data (ou tout autre dataset CSV volumineux — l'architecture reste identique)
- Créer le catalog Unity Catalog bixi_mobility avec les schémas bronze, silver, gold
- Uploader les CSV dans le volume bronze.landing_zone
- Exécuter les notebooks dans l'ordre : 01_bronze_ingestion → 02_silver_transform → 03_gold_aggregation
- (Optionnel) Configurer le Databricks Workflow pour automatiser l'enchaînement

Auteur : Daphne Fotso — Data Analyst
