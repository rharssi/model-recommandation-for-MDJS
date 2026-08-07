# MDJS — Segmentation des points de vente par clustering

Projet d'analyse non supervisée appliqué aux données de vente de MDJS (Marocaine Des Jeux et Sports) : segmentation comportementale des points de vente (retailers) par clustering, avec un module de recommandation de jeux basé sur les segments obtenus.

## Objectif

Regrouper les ~1800 retailers en segments comportementaux homogènes (volume de ventes, diversité de jeux, taux d'annulation, taux de forclusion, tendance et régularité d'activité) afin de :
1. Identifier les profils de points de vente, en particulier les segments à risque commercial élevé.
2. Recommander à chaque retailer des jeux à fort potentiel parmi ceux qu'il exploite peu ou pas encore, en s'appuyant sur leur popularité globale (voir "Moteur de recommandation" — cette recommandation n'utilise pas le clustering, malgré l'intitulé du projet).

Trois algorithmes de clustering sont comparés : K-Means, Gaussian Mixture Model (GMM), et Clustering Hiérarchique Agglomératif (Ward).

---

## Structure du projet

```
MDJS_organise/
├── data/
│   ├── data_entrainement.csv      # 1436 retailers (80%) — entraînement
│   └── data_test.csv              # 359 retailers (20%) — jamais vus à l'entraînement
├── src/                            # PIPELINE DE PRODUCTION
│   ├── common.py                      # Chargement, schéma, agrégation, préprocesseur,
│   │                                   # recommandation (popularité globale, seule méthode),
│   │                                   # seuil de risque robuste, ClusterPredictor
│   ├── decisions.py                   # Registre des décisions métier tracées (FORCED_K,
│   │                                   # RECO-METHOD-001, PROD-INDUCTIVE-001...)
│   ├── Kmeans_recommendation.py       # Pipeline K-Means (coude, PCA, profiling/risque)
│   ├── GMM_recommendation.py          # Pipeline Gaussian Mixture Model
│   ├── hiearchical_recommendation.py  # Pipeline Clustering Hiérarchique (Ward)
│   ├── Comparaison_models.py          # Compare les 3 modèles à k identique, sélectionne le meilleur
│   ├── Test_overfittinng.py           # Validation de robustesse du CLUSTERING (bootstrap,
│   │                                   # cluster à risque, bruit, hyperparamètres, train/test)
│   └── Final_recommendation.py        # Entraîne le clustering retenu (profiling/risque) +
│                                       # exporte les recommandations (popularité globale)
├── outputs/                       # Figures et métriques (généré à l'exécution de src/)
├── delivery/                      # Modèle final et recommandations (généré à l'exécution de src/)
└── requirements.txt                # Versions verrouillées (==)
```

`outputs/` et `delivery/` sont créés automatiquement au premier lancement des scripts.

**Pourquoi `common.py`** : le chargement des données, l'agrégation par retailer, le préprocesseur et la fonction de recommandation étaient auparavant dupliqués mot pour mot dans les 4 scripts de modèle. Un correctif appliqué dans l'un et oublié dans les autres était une question de temps. Ils vivent maintenant dans un seul module, importé par tous — un bugfix ou un changement de feature ne se fait plus qu'à un seul endroit. Les constantes `RANDOM_STATE` et `FORCED_K`, ainsi que le seuil de risque robuste et le `ClusterPredictor` (voir plus bas), suivent le même principe.

### Validation

```bash
pip install -r requirements.txt
cd src
python Kmeans_recommendation.py        # clustering + profiling/risque
python Comparaison_models.py           # compare les 3 algos, désigne le meilleur
python Test_overfittinng.py            # robustesse du clustering (bootstrap, bruit, train/test)
python Final_recommendation.py         # entraîne le modèle retenu, exporte les recommandations
```

---

## Installation

```bash
pip install -r requirements.txt
```

Python 3.10 ou supérieur recommandé.

## Utilisation

Depuis le dossier `src/` :

```bash
python Kmeans_recommendation.py       # entraîne et évalue K-Means seul
python GMM_recommendation.py          # entraîne et évalue GMM seul
python hiearchical_recommendation.py  # entraîne et évalue le clustering hiérarchique seul
python Comparaison_models.py          # compare les 3 modèles et sauvegarde le meilleur
python Test_overfittinng.py           # stabilité, sensibilité au bruit, validation sur data_test.csv
python Final_recommendation.py        # entraîne le modèle final et exporte les recommandations
```

Chaque module est aussi importable directement, par exemple :

```python
import Kmeans_recommendation as km
df_retailers, pivot, X, preprocessor = km.load_and_prepare()
```

---

## Données

Niveau natif : une ligne par `RETAILER_CODE` × `GAME_CODE`.

| Colonne | Description |
|---|---|
| `Total Ventes Brutes` / `Total Ventes Nettes` | Montants de ventes |
| `Total Tickets Vendus` / `Total Tickets Annulés` | Volumes de tickets |
| `Taux_Forclusion_Pct` | Taux de forclusion (numérique) |
| `TREND_SCORE` | Pente de tendance sur les ventes mensuelles |
| `AVG_MOVING_AVERAGE_3M` / `AVG_GROWTH_RATE` | Moyenne mobile 3 mois / croissance mois-à-mois |
| `SHARE_IMPUTED` | Part des mois imputés plutôt que réels dans l'historique du retailer |
| `ACTIVITY_RATE_12M`, `ACTIVITY_REGULARITY` | Fréquence et régularité d'activité sur 12 mois |

`data_entrainement.csv` et `data_test.csv` sont issus d'un split aléatoire 80/20 par `RETAILER_CODE` (jamais par ligne), garantissant qu'aucun retailer n'apparaît des deux côtés.

---

## Choix méthodologiques

**Transformation d'échelle** : les features de volume (ventes, tickets) suivent une distribution très asymétrique, avec des retailers plusieurs ordres de grandeur au-dessus de la médiane. Un `log1p` est appliqué avant le `StandardScaler` pour éviter qu'un seul point extrême ne domine le clustering. `TREND_SCORE`, pouvant être négatif, utilise un log signé (`sign(x) · log1p(|x|)`).

**Nombre de clusters (k=3)** : fixé via la constante `FORCED_K` en tête de `Kmeans_recommendation.py` et `hiearchical_recommendation.py`. Ce choix privilégie un segment métier significatif (voir Résultats) plutôt que l'optimum purement statistique (k=2). Mettre `FORCED_K = None` pour revenir à la sélection automatique par score de silhouette.

Cette décision est tracée dans `src/decisions.py` (`FORCED_K_DECISION`, id `CLUSTERING-K-001`) : optimum statistique écarté, justification métier, preuves, et champs `validated_by` / `validation_date` à faire remplir par un référent métier MDJS avant mise en production. Tant qu'ils sont vides, chaque script concerné affiche un avertissement explicite au lancement — la décision n'est pas cachée dans un commentaire de code.

**Comparaison des 3 algorithmes à k identique** : `Comparaison_models.py` évalue K-Means, GMM et le Clustering Hiérarchique tous avec k=3 (au lieu de laisser GMM libre de choisir son optimum BIC propre, n=8 en pratique — comparer un modèle contraint à un modèle libre n'aurait pas été une comparaison à conditions égales). Cette décision est tracée sous `CLUSTERING-K-002` dans `src/decisions.py`. L'optimum statistique libre de chaque modèle reste calculé et publié à titre de référence dans `comparison_results.json` (`reference_free_k_not_used_for_selection`), mais n'entre jamais dans la sélection du meilleur modèle.

**Agrégation des taux** : `taux_annulation_pct` et `taux_forclusion_pct` sont recalculés au niveau retailer par moyenne pondérée par les ventes, plutôt que moyennés ligne à ligne, pour rester statistiquement corrects.

**Qualité des données — taux d'annulation > 100%** : 6 retailers de `data_entrainement.csv` (et 1 de `data_test.csv`) ont plus de tickets annulés que de tickets vendus sur la période (probable annulation d'un ticket vendu hors fenêtre d'extraction, ou erreur source). La valeur n'est pas clippée à 100% pour ne pas masquer un signal réel : elle est conservée telle quelle et exposée via la colonne `ANOMALIE_ANNULATION_SUP_100PCT` dans `df_retailers`, avec un avertissement explicite affiché au chargement (`build_retailer_features`). A vérifier côté ERP MDJS.

**`nb_channels` retiré du clustering** : cette feature (`nunique(RETAILER_NAME)` par `RETAILER_CODE`) a été calculée mais retirée de `NUMERIC_FEATURES` — vérification faite sur `data_entrainement.csv` : seuls 5 retailers / 1436 ont plus d'un nom associé, donc la feature vaut quasi toujours 1 (variance ~nulle, aucun pouvoir discriminant pour le clustering). Elle reste calculée dans `df_retailers` à titre diagnostique (repérage d'incohérences de nommage).

**Détection de risque robuste (médiane + MAD)** : le seuil de risque commercial (`interpret_clusters()` dans les 3 pipelines) utilise `common.robust_risk_threshold()`, basé sur médiane + 3×MAD (Median Absolute Deviation) plutôt que moyenne + 1 écart-type. `taux_forclusion_pct` et `taux_annulation_pct` contiennent des valeurs extrêmes (retailers > 100% d'annulation, voir `ANOMALIE_ANNULATION_SUP_100PCT`) qui inflateraient disproportionnellement une moyenne et un écart-type classiques — cohérent avec le traitement de l'asymétrie déjà appliqué via `log1p` sur les features de volume.

**Compatibilité production des modèles transductifs** : `AgglomerativeClustering` (Clustering Hiérarchique) n'a pas de méthode `.predict()` — c'est un algorithme transductif, qui ne sait affecter que les points vus au moment du fit. Si `Comparaison_models.py` désigne "Hierarchical" comme meilleur modèle, `common.ClusterPredictor` entraîne un classifieur `NearestCentroid` en surrogate sur `(X, labels)` du clustering final, pour que le pipeline sauvegardé par `Final_recommendation.py` reste utilisable en production quel que soit l'algorithme retenu. Cette approximation (affectation par centroïde le plus proche, pas un recalcul de la hiérarchie Ward) est tracée comme décision métier dans `src/decisions.py` (`PROD_INDUCTIVE_DECISION`, id `PROD-INDUCTIVE-001`).

**Cohérence de `FORCED_K` entre modules** : `RANDOM_STATE` et `FORCED_K` vivent désormais uniquement dans `common.py` (source unique), importés par `Kmeans_recommendation.py`, `hiearchical_recommendation.py` et `Comparaison_models.py` — plus de redéfinition locale possible. `common.validate_forced_k_consistency()` reste un garde-fou explicite, appelé au chargement de `Comparaison_models.py`, qui échoue bruyamment si jamais un module venait à diverger.

**Stabilité du cluster à risque (minoritaire)** : l'ARI bootstrap global (0,972) peut masquer l'instabilité d'un petit segment (~16 retailers / 1436) noyé dans une bonne moyenne. `Test_overfittinng.py` mesure donc en plus une similarité de Jaccard spécifique au cluster à risque à travers les ré-échantillonnages bootstrap (`risk_cluster_stability()`, figure `risk_cluster_stability.png`) : Jaccard moyen 0,786, confirmant que ce segment n'est pas un artefact d'échantillonnage.

**Validation de schéma en entrée** : `common.load_raw_data()` vérifie la présence de toutes les colonnes attendues avant tout traitement et lève une `SchemaError` explicite (colonnes manquantes listées) si le CSV source ne correspond pas au format attendu, plutôt qu'un `KeyError` cryptique loin de la cause réelle.

---

## Résultats

### Segmentation (K-Means, k=3)

| Cluster | Retailers | Profil |
|---|---|---|
| 0 | 1168 | Volume élevé, multi-jeux, faible risque (forclusion 1,4%) |
| 1 | 15 | Risque commercial élevé (forclusion 56%, annulation 159%) |
| 2 | 253 | Volume moyen, profil intermédiaire |

- Silhouette : 0,496 · Davies-Bouldin : 1,01 (meilleur score des 3 modèles à k=3)
- Stabilité par bootstrap (ARI) : 0,977
- Cohérence train → test (ARI) : 0,734

### Moteur de recommandation

Une seule méthode est implémentée en production : **popularité globale** (`common.recommend_games_global()`), fixée directement dans le code — aucune sélection à l'exécution. Ce choix s'appuie sur une comparaison à 3 méthodes menée hors du pipeline de production, par validation **leave-one-out** : pour 1364/1436 retailers (≥ 2 jeux à ventes positives), un jeu déjà exploité est caché, puis chaque méthode tente de le retrouver (Mean Reciprocal Rank).

| Approche | MRR | IC bootstrap 95% |
|---|---|---|
| Popularité globale (sans segmentation) | **0,662** | [0,645, 0,679] |
| Popularité intra-cluster | **0,662** | [0,645, 0,679] |
| Score de lift (sur-représentation par cluster) | 0,477 | [0,464, 0,491] |

Popularité globale et intra-cluster obtiennent des scores **strictement identiques**, retailer par retailer (test de Wilcoxon apparié : aucune différence à tester). Le score de lift est significativement pire (p<0,0001). Explication : le catalogue est extrêmement concentré (un seul jeu = **93,5%** du volume total) — le classement des jeux par popularité reste quasi identique quel que soit le sous-groupe de retailers sur lequel on agrège, d'où l'égalité stricte ; le score de lift, qui cherche au contraire des jeux sur-représentés *dans* un cluster, fait remonter des jeux de niche rarement égaux au jeu réellement caché.

**Décision** (tracée sous `RECO-METHOD-001` dans `src/decisions.py`) : à performance strictement égale entre les 2 meilleures méthodes, la popularité globale est retenue et fixée directement dans le code — plus simple (aucune dépendance à l'affectation de cluster d'un retailer, donc rien à recalculer si le clustering change ou si un nouveau retailer arrive sans historique).

**Le clustering et la recommandation sont deux usages indépendants des mêmes données**, pas deux étapes d'un même pipeline : les 3 scripts par-algorithme (`Kmeans_recommendation.py`, `GMM_recommendation.py`, `hiearchical_recommendation.py`), `Comparaison_models.py` et `Test_overfittinng.py` s'arrêtent au **profiling et à la détection de risque commercial** — ils ne produisent aucune recommandation. `Final_recommendation.py` reste le seul point d'entrée qui fait les deux (entraîne le clustering retenu pour le risque, ET appelle `recommend_games_global()` pour la recommandation), car c'est le script livré en production.

**En résumé** : la segmentation est robuste et utile pour le profiling et la détection de risque commercial. Elle n'apporte pas, chiffres à l'appui, de gain mesurable pour la recommandation de jeux.

---

## Limites connues

- Historique mensuel réel disponible pour une partie seulement des retailers ; le reste est imputé à partir du total annuel réel et d'un profil de saisonnalité appris (voir `SHARE_IMPUTED`).
- Le catalogue de jeux est fortement concentré (93,5% du volume sur un seul jeu), ce qui limite tout système de recommandation basé sur la popularité, indépendamment de la méthode de segmentation.
- Le taux d'annulation supérieur à 100% observé sur le segment à risque mérite vérification côté données sources.
- GMM performe nettement moins bien que K-Means et le clustering hiérarchique sur ces données (silhouette ~0,19).
- Si Hierarchical est un jour retenu en production, l'affectation de nouveaux retailers passe par un surrogate NearestCentroid (approximation), pas un vrai recalcul de la hiérarchie Ward — voir `PROD-INDUCTIVE-001` dans `src/decisions.py`.
- L'évaluation MRR utilise une méthodologie leave-one-out, pas le split temporel (9 mois / 3 mois) qu'on utiliserait idéalement : les CSV fournis n'ont pas de granularité mensuelle. Le leave-one-out est une méthodologie standard pour ce type de données (feedback implicite sans horodatage), mais mesure une question légèrement différente ("retrouve-t-on un jeu déjà exploité en le cachant ?" plutôt que "prédit-on un achat futur ?").

## Pistes d'amélioration

- Enrichir avec la hiérarchie produit complète (`GAME_NAME`, `GAME_CATEGORY`) au-delà du seul `GAME_CODE`.
- Valider manuellement les retailers à région ambiguë (`REGION_WAS_AMBIGUOUS`).
- Retester le moteur de recommandation si le catalogue de jeux s'élargit ou se rééquilibre — l'égalité entre popularité globale et intra-cluster dépend directement du niveau de concentration actuel.
- Faire valider par un référent métier MDJS les décisions tracées dans `src/decisions.py` (`validated_by` / `validation_date` actuellement vides).
- Investiguer côté ERP les retailers avec `ANOMALIE_ANNULATION_SUP_100PCT = True`.
- Si des données mensuelles deviennent disponibles, refaire la comparaison des méthodes de recommandation via un vrai split temporel plutôt que le leave-one-out actuel.