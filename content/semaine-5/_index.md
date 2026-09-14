+++
title = "Semaine 5 — Tests automatisés et intégration continue"
type = "chapter"
weight = 6
draft = true
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Distinguer un test unitaire d'un test de régression et expliquer pourquoi un test qui échoue
  avant une correction a plus de valeur qu'un test qui passe du premier coup.
- Mesurer la couverture de code d'une suite de tests et interpréter ce qu'elle mesure réellement
  (et ce qu'elle ne garantit PAS).
- Appliquer des bonnes pratiques d'écriture de tests (nommage, structure Arrange-Act-Assert,
  indépendance, un seul comportement par test).
- Configurer un pipeline d'intégration continue (GitHub Actions et/ou GitLab CI) qui construit le
  projet et exécute automatiquement la suite de tests à chaque `push`/Pull Request.
- Diagnostiquer l'échec d'un pipeline CI à partir de ses journaux d'exécution.

---

## 🔍 Accroche

« Un test qui n'a jamais échoué n'a jamais rien prouvé. La vraie question n'est pas "est-ce que
mes tests passent ?", mais "est-ce que mes tests **échoueraient** si le bogue qu'ils sont censés
attraper était encore là ?". Aujourd'hui, on outille ce réflexe — et on fait en sorte qu'il
s'exécute automatiquement, à chaque `push`, sans que personne n'ait à y penser. »
