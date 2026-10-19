+++
title = "Semaine 8 — Évolution d'API : versionnage et compatibilité"
type = "chapter"
weight = 9
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Appliquer les règles de la gestion sémantique de version (*Semantic Versioning*, SemVer) pour
  décider si un changement doit incrémenter le numéro MAJEUR, MINEUR, ou de CORRECTIF.
- Distinguer un changement **compatible** (additif) d'un changement **cassant** (*breaking
  change*), selon des critères reconnus dans l'industrie.
- Reconnaître les stratégies utilisées pour signaler aux clients d'une API quelle version ils
  consomment (chemin d'URI, en-tête HTTP).
- Faire évoluer une API REST existante en lui ajoutant une fonctionnalité, **sans casser** les
  clients qui l'utilisent déjà.
- Faire correspondre un changement de code à une version Git (tag) qui en reflète fidèlement la
  nature, en réinvestissant la technique de tag vue à la semaine 7.

---

## 🔍 Accroche

« Une API, une fois publiée, devient un **contrat** : d'autres équipes — parfois des clients que
vous ne connaissez pas — écrivent du code qui dépend de sa forme exacte. Ajouter un champ, c'est
souvent sans danger. Renommer un champ, changer son type, ou le retirer, peut casser une
application que vous n'avez jamais vue tourner. Cette semaine, on apprend à nommer précisément ce
qui change (SemVer) et à distinguer ce qui peut être ajouté sans briser personne de ce qui exige
une nouvelle version. »
