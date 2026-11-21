+++
title = "Activité pratique"
weight = 4
draft = false
+++

## 🎯 Activité pratique (non notée)

En équipe, sur votre application legacy, en réinvestissant
[Gestion des bogues avec GitHub Issues]({{% relref "/semaine-10/gestion-bogues-github-issues" %}}) :

1. **Créez au moins 3 issues** sur votre dépôt GitHub, chacune correspondant à un vrai défaut
   détecté sur votre projet (par exemple une *issue* SonarCloud ou une vulnérabilité Trivy encore
   non corrigée, vues à la semaine 9).
2. **Étiquetez chaque issue** avec au moins une étiquette pertinente (`bug`, `enhancement`,
   `documentation`, ou une étiquette personnalisée comme `sécurité`) et justifiez votre choix en un
   commentaire.
3. **Créez un jalon** (ex. `TP3` ou `Correctifs semaine 10`) et associez-y vos issues, puis
   consultez la page du jalon pour voir le pourcentage d'avancement.
4. **Ouvrez une Pull Request** qui corrige l'une de ces issues, avec un mot-clé de fermeture
   automatique (`Closes #N`, `Fixes #N` ou `Resolves #N`) dans sa description, en ciblant bien la
   branche par défaut de votre dépôt.
5. **Fusionnez cette Pull Request** et confirmez que l'issue correspondante s'est bien fermée
   automatiquement.

Pour les deux issues restantes, discutez en équipe : laquelle rétroporteriez-vous en priorité si
votre projet maintenait une ancienne version en parallèle, et pourquoi (en vous appuyant sur les
critères vus dans
[Distribution de mises à jour et sécurité, section 3️⃣]({{% relref "/semaine-10/distribution-maj-securite" %}}))
?

> 💡 Cette activité clôt le cycle amorcé à la semaine 9 : détecter un problème de qualité ou de
> sécurité (SonarCloud, Trivy), le suivre formellement (GitHub Issues), puis le corriger avec une
> Pull Request qui referme la boucle automatiquement — exactement le flux de travail attendu
> pendant l'épreuve finale (semaines 13 à 15).
