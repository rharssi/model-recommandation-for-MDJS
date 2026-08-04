# model recommandation for MDJS (salers)

Pipeline de recommandation qui aide des vendeurs à identifier des produits qu'ils pourraient vendre davantage, à partir de données de vente historiques agrégées par vendeur.

## Objectif

Regrouper les vendeurs selon leur comportement commercial, puis leur recommander les produits déjà bien adoptés par les vendeurs similaires (même profil) qu'ils ne vendent pas encore eux-mêmes.

## Structure du projet

```
├── run_pipeline.py          # Point d'entrée principal
├── requirements.txt
├── src/
│   ├── config.py             # Configuration générale (colonnes, chemins, hyperparamètres)
│   ├── features.py           # Chargement, nettoyage et construction des features
│   ├── clustering.py         # Entraînement et comparaison des modèles de clustering
│   ├── recommend.py          # Génération des recommandations
│   └── evaluate.py           # Validation Leave-One-Out et comparaison à une baseline
├── data/                      # Données brutes (non versionnées, voir .gitignore)
└── results/
    ├── 1_clustering/          # Comparaison des modèles, graphiques, clusters assignés
    ├── 2_validation/          # Résultats du test Leave-One-Out
    └── 3_recommandations/     # Livrable final : recommandations par vendeur
```

## Dépendances

```bash
pip install -r requirements.txt
```

- pandas
- numpy
- scikit-learn
- matplotlib
- scipy

## Exécution

```bash
python run_pipeline.py
```

## Logique du pipeline

1. **Nettoyage des données** : chargement des ventes brutes, gestion des valeurs manquantes, conversion des taux (%) en valeurs numériques.
2. **Construction des features par vendeur** : agrégation des ventes, annulations et remboursements par vendeur, plus la part de chiffre d'affaires par produit (profil de préférence de chaque vendeur).
3. **Normalisation** des variables numériques.
4. **Clustering non supervisé** : trois modèles sont entraînés en parallèle (K-Means, Gaussian Mixture Model, Clustering hiérarchique) pour regrouper les vendeurs ayant un comportement similaire.
5. **Sélection du meilleur modèle** de clustering à partir de métriques de qualité interne (Silhouette Score, Davies-Bouldin Index, Calinski-Harabasz Index).
6. **Génération des recommandations** : pour chaque vendeur, identification des produits les plus adoptés par les autres vendeurs de son cluster qu'il ne vend pas encore.
7. **Validation honnête (Leave-One-Out)** : on cache un produit réellement acheté par chaque vendeur testable et on vérifie si le modèle le retrouve dans ses recommandations, en comparant à une baseline simple (recommander les produits les plus populaires, sans clustering).
8. **Export des résultats** dans des fichiers CSV.



