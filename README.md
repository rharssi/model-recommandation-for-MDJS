# model-recommandation-for-MDJS


## Dépendances

```bash
pip install -r requirements.txt
```

- pandas
- numpy
- scikit-learn
- matplotlib
- scipy

## Logique du pipeline

1. **Nettoyage des données** : chargement des ventes brutes, gestion des valeurs manquantes, conversion des taux (%) en valeurs numériques.
2. **Construction des features par vendeur** : agrégation des ventes, annulations et remboursements par vendeur, plus la part de chiffre d'affaires par jeu (profil de préférence).
3. **Normalisation** des variables numériques.
4. **Clustering non supervisé** : trois modèles sont entraînés en parallèle (K-Means, Gaussian Mixture Model, Clustering hiérarchique) pour regrouper les vendeurs ayant un comportement similaire.
5. **Sélection du meilleur modèle** de clustering à partir de métriques de qualité interne (Silhouette, Davies-Bouldin, Calinski-Harabasz).
6. **Génération des recommandations** : pour chaque vendeur, on identifie les jeux les plus adoptés par les autres vendeurs de son cluster qu'il ne vend pas encore.
7. **Validation honnête** : on cache un jeu réellement acheté par chaque vendeur testable et on vérifie si le modèle le retrouve dans ses recommandations, en comparant à une baseline simple (recommander les jeux les plus populaires, sans clustering).
8. **Export des résultats** dans des fichiers CSV.

## Exécution

```bash
python run_pipeline.py
```
