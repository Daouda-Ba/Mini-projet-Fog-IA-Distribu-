```markdown
#  Kafka & Spark – Mini Projet Federated Learning (Edge–Fog–Cloud)

##  Objectif du projet

Ce projet a pour objectif de mettre en œuvre une **architecture distribuée de type Edge–Fog–Cloud**
en utilisant **Apache Kafka** et **Apache Spark Structured Streaming**.

Le système simule :
- des **capteurs IoT** (Edge) produisant des données,
- des **Fog Nodes** traitant localement les données,
- un **Cloud Aggregator** construisant un modèle global à partir des résultats des Fog Nodes.

---

##  Architecture globale

```

Capteurs (Kafka Producer)
↓
Kafka Topics (raw data)
↓
Fog Node 1 (Spark Streaming) ┐
Fog Node 2 (Spark Streaming) ├──> Kafka (model-weights)
↓
Cloud Aggregator (Spark Streaming)

```

---

##  Technologies utilisées

- **Apache Kafka** (message broker)
- **Apache Spark 3.5.1** (Structured Streaming)
- **Docker & Docker Compose**
- **Python (PySpark)**
- **Zookeeper**
- **WSL2 / Windows**

---

##  Structure du projet

```

kafka-spark-federated/
│
├── docker-compose.yml
│
├── producer/
│   └── sensor_producer.py
│
├── fog/
│   ├── fog_node_1.py
│   └── fog_node_2.py
│
├── cloud/
│   └── aggregator.py
│
└── README.md

````

---

##  Phase 0 – Mise en place de l’environnement

### Lancement des services Docker

```bash
docker compose up -d
````

### Vérification des conteneurs

```bash
docker ps
```

Conteneurs attendus :

* `zookeeper`
* `kafka`
* `spark`

---

##  Phase 1 – Kafka (Edge Layer)

### Création des topics Kafka

```bash
docker exec -it kafka kafka-topics \
  --bootstrap-server kafka:29092 \
  --create --topic sensor-data \
  --partitions 1 --replication-factor 1
```

```bash
docker exec -it kafka kafka-topics \
  --bootstrap-server kafka:29092 \
  --create --topic model-weights \
  --partitions 1 --replication-factor 1
```

### Producteur de données capteurs

```bash
docker exec -it kafka python /app/producer/sensor_producer.py
```

**Données envoyées (exemple)** :

```json
{
  "sensor_id": "s1",
  "temperature": 26.5,
  "vibration": 0.03
}
```

---

##  Phase 2 – Fog Nodes (Traitement local)

### Objectif

* Consommer les données `sensor-data`
* Calculer des moyennes locales
* Générer un **poids (weight)**
* Publier vers le topic `model-weights`

---

### Commande d’exécution Fog Node

```bash
docker exec -it spark /opt/spark/bin/spark-submit \
  --conf spark.jars.ivy=/tmp/ivy \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1 \
  /app/fog/fog_node_1.py
```

Même commande pour `fog_node_2.py`.

---

###  Erreur rencontrée n°1 : Kafka non trouvé

**Erreur**

```
Failed to find data source: kafka
```

**Cause**

> Le connecteur Kafka n’est pas inclus par défaut dans Spark.

**Solution**

> Ajout du package Kafka au moment du `spark-submit`.

```bash
--packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1
```

---

###  Erreur rencontrée n°2 : Ivy cache permission denied

**Erreur**

```
FileNotFoundException: /home/spark/.ivy2/cache
```

**Solution**

> Définir un dossier Ivy accessible.

```bash
--conf spark.jars.ivy=/tmp/ivy
```

---

###  Erreur rencontrée n°3 : NoneType dans le calcul

**Erreur**

```
TypeError: unsupported operand type(s) for *: 'int' and 'NoneType'
```

**Cause**

> Certaines valeurs Kafka sont absentes (null).

**Solution**

```python
avg_temp = row.avg_temp or 0
avg_vib = row.avg_vib or 0
weight = (avg_temp + 100 * avg_vib) / 2
```

---

##  Phase 3 – Cloud Aggregator

### Objectif

* Consommer les poids envoyés par les Fog Nodes
* Calculer un **modèle global**
* Afficher le résultat en streaming

---

### Lancement du Cloud Aggregator

```bash
docker exec -it spark /opt/spark/bin/spark-submit \
  --conf spark.jars.ivy=/tmp/ivy \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1 \
  /app/cloud/aggregator.py
```

---

### Résultat attendu

```text
+-------------+
|global_weight|
+-------------+
|    56.42    |
+-------------+
```

---

##  Résultats obtenus

* ✔ Pipeline Kafka fonctionnel
* ✔ Traitement temps réel avec Spark
* ✔ Agrégation distribuée Fog → Cloud
* ✔ Architecture Federated Learning simulée
* ✔ Projet stable et reproductible

---

##  Bonnes pratiques appliquées

* Séparation Edge / Fog / Cloud
* Gestion des valeurs nulles
* Utilisation de Docker pour la reproductibilité
* Logs Spark contrôlés (`WARN`)
* Topics Kafka dédiés par couche

---

##  Améliorations possibles

* Checkpointing Spark Streaming
* Fenêtrage temporel (`window`)
* Stockage HDFS / S3
* Visualisation (Grafana)
* Sécurité Kafka (SASL / SSL)

---

## Auteur

**Daouda Ba**
Projet Kafka & Spark – Federated Learning
2026

```

---