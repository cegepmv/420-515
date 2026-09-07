+++
title = "Semaine 3 — Débogage outillé et Maven"
type = "chapter"
weight = 4
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Utiliser un débogueur pas à pas (breakpoints, *step over/into/out*, inspection des variables).
- Poser un breakpoint conditionnel pour cibler une itération précise sans tout parcourir manuellement.
- Remplacer un `System.out.println` de débogage par une journalisation structurée, avec le bon niveau.
- Appliquer une recherche par bissection (« diviser pour localiser ») pour isoler la cause d'un bogue,
  dans le code comme dans l'historique Git (`git bisect`).
- Lire et modifier un `pom.xml` : coordonnées Maven, dépendances et leur *scope*, plugins.
- Expliquer les grandes étapes du cycle de vie Maven (`validate` → ... → `install`) et utiliser
  `mvn dependency:tree` pour diagnostiquer un conflit de versions.

---

## 🔍 Accroche

« La semaine dernière, vous avez appris la **méthode** : reproduire, isoler, formuler une
hypothèse. Cette semaine, on ajoute les **outils** qui rendent cette méthode rapide en pratique —
et on ouvre le capot d'un projet Java réel : comment `mvn` sait quoi compiler, quoi tester, et
d'où viennent toutes ces dépendances dans votre `pom.xml`. »
