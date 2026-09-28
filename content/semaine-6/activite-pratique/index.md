+++
title = "Activité pratique"
weight = 3
draft = false
+++

## 🎯 Activité pratique (non notée)

En équipe, réalisez le travail suivant sur votre dépôt, via une vraie Pull/Merge Request, en
réinvestissant directement la démarche vue dans
[Réusinage (refactoring) et code smells]({{% relref "/semaine-6/refactoring" %}}) et pratiquée dans
l'[exercice Réusinage]({{% relref "/semaine-6/exercice-refactoring" %}}) :

1. **Choisissez un module entier** de votre application legacy (pas une seule méthode isolée) qui
   présente plusieurs *code smells* liés entre eux — par exemple une classe qui mélange à la fois
   une méthode trop longue, de la duplication et une liste de paramètres trop longue.
2. **Confirmez la couverture de tests** de ce module avant de commencer (voir semaine 5). S'il
   manque des tests sur une partie du comportement que vous comptez réusiner, ajoutez-les d'abord.
3. **Réusinage par étapes** : appliquez au moins **deux réusinages catalogués différents** (ex.
   *Extract Method* puis *Introduce Parameter Object*), chacun dans son propre commit, avec la
   suite de tests verte après chaque étape.
4. **Confirmez, par le pipeline CI** (semaine 5), que la suite de tests reste entièrement verte sur
   votre branche après l'ensemble des réusinages.

Pour la Pull/Merge Request :

- Une description claire de chaque réusinage appliqué (nom du réusinage, avant/après en bref) et du
  *code smell* qu'il corrige.
- Confirmation que le nombre de tests et leurs résultats sont identiques avant et après (aucun
  changement de comportement observable).
- Au moins une revue de code croisée entre équipes avant la fusion, avec une attention particulière
  à repérer si un changement de comportement se serait glissé dans le réusinage.

> 💡 Cette activité réinvestit directement les notions et l'exercice de la semaine : elle ne
> demande rien de nouveau, seulement d'appliquer la même discipline (petites étapes, tests verts,
> commits séparés) à une échelle un peu plus grande qu'un seul smell isolé.

> 📌 Cette activité **prépare** le TP2 noté (10 points), dont la remise est prévue cette semaine
> selon le plan de cours (réusinage d'un module de l'application legacy, appuyé par la suite de
> tests existante) — ce n'est pas la remise elle-même. Reportez-vous aux consignes détaillées du
> TP2 fournies séparément par l'enseignant·e pour les critères d'évaluation exacts et la date
> limite.
