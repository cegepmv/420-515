+++
title = "Semaine 4 — Maven (2/2) et introduction aux tests automatisés"
type = "chapter"
weight = 5
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Exécuter les tests d'un projet avec `mvn test`, cibler un test précis avec `-Dtest`, et lire un
  rapport Surefire.
- Distinguer les tests exécutés par `maven-surefire-plugin` (tests unitaires, phase `test`) de ceux
  exécutés par `maven-failsafe-plugin` (tests d'intégration, phase `verify`).
- Ignorer volontairement l'exécution des tests (`-DskipTests` vs `-Dmaven.test.skip`) et expliquer
  la différence entre les deux.
- Écrire une première classe de test JUnit 6 (Jupiter) : structure `@Test`, assertions de base, nommage clair.
- Regrouper plusieurs assertions liées avec `assertAll`, et vérifier qu'une exception est bien
  lancée avec `assertThrows`.
- Écrire un test paramétré (`@ParameterizedTest`) pour couvrir plusieurs cas sans dupliquer de code.
- Expliquer, sur un exemple concret du projet, pourquoi et comment un mock (Mockito) isole le code
  testé d'une dépendance difficile à construire.
- Utiliser le cycle de vie d'une classe de test (`@BeforeEach`/`@AfterEach`) pour préparer un état
  propre avant chaque test.

---

## 🔍 Accroche

« La semaine dernière, vous avez appris à lire un `pom.xml` et à suivre le cycle de vie Maven.
Cette semaine, on s'attarde sur une seule phase de ce cycle — `test` — parce que c'est probablement
celle que vous allez exécuter le plus souvent, chaque jour, pour le reste de votre carrière. Et on
écrit vos tout premiers tests JUnit sur l'application legacy : le début d'un filet de sécurité qui
va vous accompagner pour le reste de la session. »
