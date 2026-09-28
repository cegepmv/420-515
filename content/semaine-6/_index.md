+++
title = "Semaine 6 — Réusinage (refactoring) et code smells"
type = "chapter"
weight = 7
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Définir le réusinage (*refactoring*) selon Martin Fowler et expliquer en quoi il se distingue
  d'une correction de bogue ou de l'ajout d'une fonctionnalité.
- Expliquer pourquoi une suite de tests automatisés (semaine 5) est la condition préalable qui rend
  le réusinage sécuritaire, plutôt qu'une simple bonne pratique parallèle.
- Reconnaître, dans du code réel, les *code smells* les plus courants : méthode trop longue, classe
  trop grosse, duplication de code, liste de paramètres trop longue, obsession des primitifs,
  *feature envy*, *data clumps*, enchaînements conditionnels répétés.
- Appliquer un sous-ensemble du catalogue de réusinages de Fowler (extraire une méthode, extraire
  une variable, renommer, extraire une classe, remplacer un nombre magique par une constante,
  introduire un objet paramètre, remplacer une condition par du polymorphisme) à l'aide des outils
  de réusinage automatisé de l'IDE.
- Réusiner par petites étapes réversibles, en relançant la suite de tests après chaque étape, sans
  jamais mélanger un changement de comportement avec un changement de structure.
- Reconnaître les situations où il vaut mieux **reporter** un réusinage (absence de tests,
  échéance serrée, code sur le point d'être supprimé).

---

## 🔍 Accroche

« Vos tests passent, votre pipeline CI est vert (semaine 5) — le code **fonctionne**. Mais
"fonctionner" et "être facile à faire évoluer" sont deux choses différentes. Cette semaine, on
s'attaque à la structure interne du code, sans jamais changer ce qu'il fait de l'extérieur. Le
filet de sécurité que vous avez tissé la semaine dernière n'était pas un exercice isolé : c'est
précisément ce qui va vous permettre de réusiner sans retenue, en sachant immédiatement si vous
avez cassé quelque chose. »
