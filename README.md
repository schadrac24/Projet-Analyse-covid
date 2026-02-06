## 1. Présentation du dataset

### 1.1 Description générale

| Caractéristique | Valeur |
|-----------------|--------|
| **Source** | Public Health Agency of Canada - Données COVID-19 |
| **Format** | CSV |
| **Taille** | 76 MB |
| **Nombre de lignes** | 903 597 |
| **Nombre de colonnes** | 13 |

Le dataset répond aux critères de l'examen :
- Minimum 500 000 lignes : **903 597 lignes**
- Minimum 5 colonnes : **13 colonnes**
- Format accepté : **CSV**

### 1.2 Structure des données

| Colonne | Type | Description |
|---------|------|-------------|
| PEOPLE_POSITIVE_CASES_COUNT | Integer | Nombre cumulatif de cas positifs |
| COUNTY_NAME | String | Nom du comté/région |
| PROVINCE_STATE_NAME | String | Province ou État |
| REPORT_DATE | Date | Date du rapport (format YYYY-MM-DD) |
| CONTINENT_NAME | String | Nom du continent |
| DATA_SOURCE_NAME | String | Source des données |
| PEOPLE_DEATH_NEW_COUNT | Integer | Nouveaux décès du jour |
| COUNTY_FIPS_NUMBER | String | Code FIPS du comté |
| COUNTRY_ALPHA_3_CODE | String | Code pays ISO 3 lettres |
| COUNTRY_SHORT_NAME | String | Nom court du pays |
| COUNTRY_ALPHA_2_CODE | String | Code pays ISO 2 lettres |
| PEOPLE_POSITIVE_NEW_CASES_COUNT | Integer | Nouveaux cas positifs du jour |
| PEOPLE_DEATH_COUNT | Integer | Nombre cumulatif de décès |

### 1.3 Contexte

Ce dataset compile les données quotidiennes de la pandémie de COVID-19 à l'échelle mondiale. Il permet d'analyser :
- L'évolution temporelle de la pandémie
- La distribution géographique des cas et décès
- Les différences entre pays et continents
- L'impact spécifique au Canada par province

---

## 2. Choix techniques et modélisation

### 2.1 Framework : Apache Spark

Nous avons choisi **Apache Spark** plutôt que Hadoop MapReduce pour les raisons suivantes :

| Critère | Spark | MapReduce |
|---------|-------|-----------|
| Performance | Jusqu'à 100x plus rapide (in-memory) | Lecture/écriture disque à chaque étape |
| API | Python (PySpark) intuitif | Java principalement, plus verbeux |
| Interactivité | Compatible Jupyter Notebook | Batch uniquement |
| Courbe d'apprentissage | Modérée | Élevée |

### 2.2 Structure de données : DataFrames

Nous avons opté pour les **DataFrames** plutôt que les RDD :

**Avantages des DataFrames :**
1. **Catalyst Optimizer** : Optimisation automatique des requêtes
2. **Tungsten Engine** : Gestion mémoire optimisée
3. **API déclarative** : Syntaxe proche de SQL et Pandas
4. **Inférence de schéma** : Détection automatique des types de données
5. **Meilleures performances** : Sérialisation optimisée

### 2.3 Architecture technique

```
┌─────────────────────────────────────────────────────────────────┐
│                      DOCKER COMPOSE                              │
├───────────────────┬───────────────────┬─────────────────────────┤
│   SPARK MASTER    │   SPARK WORKER    │    JUPYTER NOTEBOOK     │
│   (Port 8080)     │                   │    (Port 8888)          │
│   apache/spark    │   apache/spark    │    jupyter/pyspark      │
│   :3.5.0          │   :3.5.0          │    -notebook            │
├───────────────────┴───────────────────┴─────────────────────────┤
│                      VOLUMES PARTAGÉS                            │
│   /data (dataset)  │  /notebooks (code)  │  /output (résultats) │
└─────────────────────────────────────────────────────────────────┘
```

Cette architecture permet :
- Un environnement reproductible via Docker
- La simulation d'un cluster distribué
- L'analyse interactive via Jupyter
- Le partage automatique des fichiers entre conteneurs et Windows

---

## 3. Préparation des données

### 3.1 Ingestion des données

```python
df = spark.read.csv(
    "/home/jovyan/data/covid_data.csv",
    header=True,
    inferSchema=True
)
```

- **header=True** : Première ligne comme en-têtes
- **inferSchema=True** : Détection automatique des types

### 3.2 Vérification de la qualité

**Analyse des valeurs manquantes :**

| Colonne | Valeurs nulles |
|---------|----------------|
| COUNTY_NAME | 858 196 |
| COUNTY_FIPS_NUMBER | 858 196 |
| Autres colonnes | 0 |

Les colonnes COUNTY sont majoritairement nulles car les données sont agrégées au niveau province/pays.

### 3.3 Nettoyage effectué

1. **Sélection des colonnes pertinentes** : 8 colonnes sur 13 retenues
2. **Conversion de date** : Transformation en type DateType
3. **Extraction temporelle** : Création des colonnes YEAR et MONTH
4. **Filtrage** : Suppression des lignes avec pays ou cas nulls

**Résultat du nettoyage :**
- Lignes initiales : 903 597
- Lignes après nettoyage : 903 596
- Lignes supprimées : 1

---

## 4. Transformations et actions

### 4.1 Transformations utilisées (6)

| # | Transformation | Syntaxe | Utilisation dans le projet |
|---|---------------|---------|---------------------------|
| 1 | `select()` | `df.select("col1", "col2")` | Réduction aux 8 colonnes pertinentes |
| 2 | `filter()` | `df.filter(condition)` | Suppression des nulls, filtre Canada |
| 3 | `withColumn()` | `df.withColumn("new", expr)` | Création YEAR, MONTH, taux mortalité |
| 4 | `groupBy()` | `df.groupBy("col")` | Agrégation par pays, continent, province |
| 5 | `orderBy()` | `df.orderBy(desc("col"))` | Tri décroissant par nombre de cas |
| 6 | `agg()` | `df.agg(sum(), max(), count())` | Calculs d'agrégation multiples |

### 4.2 Actions utilisées (3)

| # | Action | Syntaxe | Utilisation dans le projet |
|---|--------|---------|---------------------------|
| 1 | `count()` | `df.count()` | Comptage des lignes pour validation |
| 2 | `show()` | `df.show(n)` | Affichage des résultats d'analyse |
| 3 | `take()` / `collect()` | `df.take(n)` | Récupération pour traitement Python |

### 4.3 Exemple de code

```python
# Exemple : Top 10 pays les plus touchés
top_10_pays = df_covid \
    .groupBy("COUNTRY_SHORT_NAME") \
    .agg(
        max("PEOPLE_POSITIVE_CASES_COUNT").alias("TOTAL_CAS"),
        max("PEOPLE_DEATH_COUNT").alias("TOTAL_DECES")
    ) \
    .orderBy(desc("TOTAL_CAS")) \
    .limit(10)
```

---

## 5. Questions d'analyse et résultats

### Question 1 : Quels sont les 10 pays les plus touchés ?

**Méthodologie** : Agrégation par pays du maximum de cas cumulatifs, tri décroissant.

**Résultats :**

| Rang | Pays | Cas totaux | Décès | Taux mortalité |
|------|------|------------|-------|----------------|
| 1 | Inde | 6 623 815 | 102 685 | 1.55% |
| 2 | Brésil | 4 915 289 | 146 352 | 2.98% |
| 3 | Russie | 1 215 001 | 21 358 | 1.76% |
| 4 | Colombie | 855 052 | 26 712 | 3.12% |
| 5 | Pérou | 828 169 | 32 742 | 3.95% |
| 6 | Argentine | 798 473 | 21 018 | 2.63% |
| 7 | Espagne | 789 932 | 32 086 | 4.06% |
| 8 | Mexique | 761 665 | 79 088 | 10.38% |
| 9 | Afrique du Sud | 681 289 | 16 976 | 2.49% |
| 10 | France | 619 170 | 32 230 | 5.21% |

**Interprétation** : Les pays les plus touchés sont des grandes économies à forte population. Le Mexique présente un taux de mortalité exceptionnellement élevé (10.38%), possiblement dû à des sous-déclarations de cas ou des problèmes d'accès aux soins.

![Top 10 pays](output/graphique_top10_pays.png)

---

### Question 2 : Comment évoluent les cas dans le temps ?

**Méthodologie** : Agrégation mensuelle des nouveaux cas et décès.

**Résultats (extrait) :**

| Période | Nouveaux cas | Nouveaux décès |
|---------|--------------|----------------|
| Déc 2019 | 27 | 0 |
| Jan 2020 | 9 799 | 213 |
| Fév 2020 | 74 706 | 2 703 |
| Mar 2020 | 748 875 | 37 019 |
| Avr 2020 | 2 340 248 | 190 140 |
| Mai 2020 | 2 863 785 | 138 445 |
| Juin 2020 | 4 262 901 | 134 593 |
| Juil 2020 | 7 058 361 | 166 558 |
| Août 2020 | 7 932 048 | 176 788 |
| Sept 2020 | 8 462 925 | 162 137 |

**Interprétation** : On observe une croissance exponentielle des cas entre mars et septembre 2020, avec un pic en septembre (8.4 millions de nouveaux cas). La mortalité suit une tendance similaire avec un décalage temporel.

![Évolution temporelle](output/graphique_evolution_temporelle.png)

---

### Question 3 : Quelle est la distribution par continent ?

**Méthodologie** : Agrégation par continent avec calcul du taux de mortalité.

**Résultats :**

| Continent | Cas totaux | Décès | Taux mortalité |
|-----------|------------|-------|----------------|
| Asie | 6 623 815 | 102 685 | 1.55% |
| Amérique | 4 915 289 | 146 352 | 2.98% |
| Europe | 1 215 001 | 42 350 | 3.49% |
| Afrique | 681 289 | 16 976 | 2.49% |
| Océanie | 27 136 | 894 | 3.29% |

**Interprétation** : L'Asie concentre le plus de cas (dominé par l'Inde), mais l'Europe présente le taux de mortalité le plus élevé (3.49%), possiblement dû à une population plus âgée.

![Distribution continents](output/graphique_continents.png)

---

### Question 4 : Quelles provinces canadiennes sont les plus touchées ?

**Méthodologie** : Filtrage Canada, agrégation par province.

**Résultats :**

| Province | Cas totaux | Décès | Taux mortalité |
|----------|------------|-------|----------------|
| Québec | 78 459 | 5 878 | 7.49% |
| Ontario | 54 199 | 2 975 | 5.49% |
| Alberta | 18 357 | 272 | 1.48% |
| Colombie-Britannique | 9 381 | 238 | 2.54% |
| Manitoba | 2 140 | 23 | 1.07% |

**Interprétation** : Le Québec et l'Ontario concentrent 80% des cas canadiens. Le Québec présente un taux de mortalité élevé (7.49%), particulièrement dans les CHSLD durant la première vague.

![Provinces Canada](output/graphique_provinces_canada.png)

---

## 6. Problèmes techniques rencontrés

### 6.1 Problème d'image Docker

**Erreur :**
```
Error response from daemon: failed to resolve reference 
"docker.io/bitnami/spark:3.5": not found
```

**Cause** : L'image `bitnami/spark:3.5` n'existe pas sur Docker Hub.

**Solution** : Utiliser l'image officielle Apache Spark :
```yaml
image: apache/spark:3.5.0
```

---

### 6.2 Incompatibilité PySpark / Java

**Erreur :**
```
TypeError: 'JavaPackage' object is not callable
```

**Cause** : PySpark 4.0.1 (installé par défaut) est incompatible avec la version Java du conteneur.

**Solution** : Installer une version compatible de PySpark :
```python
pip install pyspark==3.5.0
```

Nous avons automatisé cette installation dans le script `run.py` :
```python
def check_and_install_dependencies():
    required_packages = [("pyspark", "3.5.0"), ...]
    for package, version in required_packages:
        try:
            __import__(package)
        except ImportError:
            subprocess.check_call([sys.executable, "-m", "pip", 
                                  "install", f"{package}=={version}"])
```

---

### 6.3 Conflit de fonction max()

**Erreur :**
```
TypeError: max() takes 1 positional argument but 2 were given
```

**Cause** : Conflit entre la fonction Python `max()` et la fonction Spark `max()` lors de la génération des graphiques.

**Solution** : Utiliser `len()` avec une condition pour éviter le conflit :
```python
# Avant (erreur)
colors = plt.cm.Blues(range(50, 250, int(200/max(len(df_pd), 1))))

# Après (corrigé)
num_provinces = len(df_pd) if len(df_pd) > 0 else 1
colors = plt.cm.Blues([i/num_provinces for i in range(num_provinces)])
```

---

## 7. Optimisations

### 7.1 Mise en cache

```python
df_covid.cache()
```

Le DataFrame nettoyé est mis en cache en mémoire pour éviter de recharger les données à chaque requête.

### 7.2 Réduction des partitions

```python
df.coalesce(1).write.csv(...)
```

Utilisation de `coalesce(1)` avant l'écriture pour générer un seul fichier CSV au lieu de multiples fichiers partitionnés.

### 7.3 Sélection précoce

Les colonnes non nécessaires sont éliminées dès le début du pipeline pour réduire le volume de données en mémoire.

### 7.4 Filtrage anticipé

Les filtres sont appliqués avant les agrégations coûteuses pour minimiser le nombre de lignes traitées.

---

## 8. Conclusions et limites

### 8.1 Résultats clés

1. **L'Inde et le Brésil** dominent en nombre de cas absolus
2. **Le Mexique** présente un taux de mortalité alarmant (10.38%)
3. **L'Europe** a le taux de mortalité continental le plus élevé
4. **Au Canada**, le Québec concentre le plus de cas et le plus haut taux de mortalité

### 8.2 Limites de l'analyse

| Limite | Impact |
|--------|--------|
| Méthodes de comptage variables | Comparaisons internationales biaisées |
| Données non normalisées par population | Pays peuplés surreprésentés |
| Période limitée (fin 2020) | Vision partielle de la pandémie |
| Sous-déclaration probable | Chiffres réels potentiellement plus élevés |

### 8.3 Pistes d'amélioration

1. **Normaliser par population** : Cas pour 100 000 habitants
2. **Croiser avec d'autres données** : PIB, densité, âge médian
3. **Étendre la période** : Inclure les vagues suivantes
4. **Modèles prédictifs** : Utiliser Spark MLlib pour des prédictions

---
