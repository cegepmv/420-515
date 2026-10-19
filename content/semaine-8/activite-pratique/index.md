+++
title = "Activité pratique"
weight = 5
draft = false
+++

## 🎯 Activité pratique (non notée)

En équipe, sur votre application legacy, en réinvestissant directement
[SemVer]({{% relref "/semaine-8/semver" %}}) et
[Compatibilité d'API]({{% relref "/semaine-8/compatibilite-api" %}}) :

1. **Ajoutez une fonctionnalité additive** à votre API REST (ex. un nouveau champ optionnel dans
   une réponse existante, ou un nouvel endpoint) — un changement qui, selon AIP-180 (voir
   [Compatibilité d'API]({{% relref "/semaine-8/compatibilite-api" %}})), ne casse **aucun** client
   existant.
2. **Validez** que les clients existants (vos tests automatisés de la semaine 5) continuent de
   passer sans aucune modification — c'est la preuve concrète que le changement est bien
   rétrocompatible.
3. **Mettez à jour le numéro de version** de votre projet (`pom.xml`) selon la règle de SemVer
   applicable à ce type de changement (voir [SemVer]({{% relref "/semaine-8/semver" %}}), règle 7).
4. **Créez et poussez un tag Git** correspondant à cette nouvelle version, sur le modèle vu dans
   [Tags Git et publications (Releases) GitHub]({{% relref "/semaine-8/tags-releases-github" %}}) :

   ```bash
   git tag v<votre-nouvelle-version>
   git push origin v<votre-nouvelle-version>
   ```

5. Si votre pipeline CI publie déjà des Releases GitHub par tag, confirmez que la
   Release correspondant à ce tag apparaît bien dans l'onglet **Releases** de votre dépôt.

Pour la Pull Request :

- Le changement additif apporté à l'API, avec sa justification (pourquoi il est compatible, selon
  les critères d'AIP-180 vus cette semaine).
- Le numéro de version mis à jour dans `pom.xml`, avec le type d'incrément justifié (MAJEUR,
  MINEUR, ou CORRECTIF).
- Le tag Git poussé correspondant à cette version.
