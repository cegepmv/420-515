+++
title = "Activité pratique"
weight = 6
draft = false
+++

## 🎯 Activité pratique (non notée)

En équipe, réalisez le travail suivant sur votre dépôt, via une vraie Pull Request :

1. **Ajoutez une suite de tests** à votre application legacy : au moins un test unitaire et un test
   de régression (voir [Tests automatisés]({{% relref "/semaine-5/tests-automatises" %}})), avec
   une couverture mesurée via JaCoCo pour au moins un module de votre choix.
2. **Intégrez cette suite à un pipeline CI** (GitHub Actions) qui l'exécute automatiquement à
   chaque `push`/Pull Request (voir
   [Intégration continue]({{% relref "/semaine-5/integration-continue" %}})).
3. Confirmez, par une capture ou un lien, que le pipeline est bien **vert** sur votre branche
   principale.

Pour la Pull Request :

- Une description claire des tests ajoutés (unitaire vs régression, et pourquoi).
- Le fichier de configuration CI inclus dans la même Pull Request (ou une PR séparée liée).
- Au moins une revue de code croisée entre équipes avant la fusion.
