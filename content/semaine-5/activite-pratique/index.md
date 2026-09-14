+++
title = "Activité pratique"
weight = 5
draft = true
+++

## 🎯 Activité pratique (non notée)

En équipe, réalisez le travail suivant sur votre dépôt, via une vraie Pull/Merge Request :

1. **Ajoutez une suite de tests** à votre application legacy : au moins un test unitaire et un test
   de régression (voir [Tests automatisés]({{% relref "/semaine-5/tests-automatises" %}})), avec
   une couverture mesurée via JaCoCo pour au moins un module de votre choix.
2. **Intégrez cette suite à un pipeline CI** (GitHub Actions ou GitLab CI) qui l'exécute
   automatiquement à chaque `push`/Pull Request (voir
   [Intégration continue]({{% relref "/semaine-5/integration-continue" %}})).
3. Confirmez, par une capture ou un lien, que le pipeline est bien **vert** sur votre branche
   principale.

Pour la Pull/Merge Request :

- Une description claire des tests ajoutés (unitaire vs régression, et pourquoi).
- Le fichier de configuration CI inclus dans la même Pull/Merge Request (ou une PR séparée liée).
- Au moins une revue de code croisée entre équipes avant la fusion.

> 💡 Cette activité réinvestit directement les deux exercices de la semaine : elle ne demande rien
> de nouveau, seulement d'assembler ce que vous avez déjà pratiqué (tests + pipeline) en un
> livrable cohérent pour votre équipe.

> 📌 Cette activité **prépare** le TP2 noté (10 points), prévu à la **semaine 6** selon le plan de
> cours (réusinage appuyé sur la suite de tests que vous construisez ici) — ce n'est pas la remise
> elle-même. Aucun quiz n'est prévu à la semaine 5.
