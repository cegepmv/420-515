+++
title = "Semaine 2 — Lire le code des autres"
type = "chapter"
weight = 3
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Appliquer des stratégies de lecture de code (top-down vs bottom-up) pour comprendre un programme
  écrit par quelqu'un d'autre.
- Utiliser les tests existants comme documentation vivante.
- Repérer des indices de code smells à l'aide d'une grille de lecture simple.
- Reproduire et isoler un bogue de façon fiable avant de tenter de le corriger.
- Lire une pile d'appels (stack trace) pour localiser la cause d'une exception, y compris sa
  cause racine (`Caused by:`).
- Formuler et tester des hypothèses de cause, plutôt que de corriger « à l'aveugle ».
- Ouvrir et gérer une Pull Request, effectuer une revue de code.
- Distinguer `merge` et `rebase`, ainsi que `revert` et `reset`.
- Utiliser `git cherry-pick` pour appliquer un commit précis d'une branche à une autre.

---

## 🔍 Accroche

« N'importe qui peut écrire du code. Peu de gens savent le LIRE efficacement. C'est une compétence
distincte, presque un sixième sens qu'on développe. Aujourd'hui, on l'entraîne. »
