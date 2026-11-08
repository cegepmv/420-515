+++
title = "Semaine 9 — Qualité et sécurité du code : SonarQube/SonarCloud et Trivy"
type = "chapter"
weight = 10
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Expliquer ce qu'est l'**analyse statique** de code et en quoi elle diffère des tests
  automatisés (semaine 4) : elle examine le code **sans l'exécuter**.
- Distinguer les trois **qualités logicielles** évaluées par SonarQube/SonarCloud — sécurité,
  fiabilité, maintenabilité — et les types de règles associés (bug, vulnérabilité, *code smell*,
  *security hotspot*).
- Lire et interpréter un **quality gate** (porte de qualité) SonarCloud pour décider si un code
  est prêt à être fusionné ou livré.
- Intégrer une analyse SonarCloud dans un pipeline GitHub Actions existant (semaine 5), avec
  couverture de tests transmise via JaCoCo.
- Expliquer ce qu'est une **vulnérabilité** (CVE) dans une dépendance ou une image Docker, et
  pourquoi ce risque est distinct des défauts de code trouvés par SonarQube.
- Intégrer une analyse de vulnérabilités **Trivy** dans le même pipeline CI/CD, et faire échouer
  intentionnellement le build sur des vulnérabilités critiques.
- **Corriger** de vrais enjeux de qualité et de sécurité détectés par ces outils sur l'application
  legacy du cours.

---

## 🔍 Accroche

« Votre pipeline CI (semaine 5) répond déjà à une question : *est-ce que ça compile, et les tests
passent-ils ?* Mais un code peut compiler, passer tous ses tests, et quand même être truffé de
duplications, de méthodes à complexité démesurée, ou pire : dépendre d'une bibliothèque qui
contient une faille de sécurité **connue publiquement**, exploitable par n'importe qui en
consultant une base de données ouverte. Cette semaine, deux outils automatisés s'ajoutent à votre
pipeline pour répondre à ces deux questions que les tests ne posent jamais : *ce code est-il
maintenable ?* et *ce code expose-t-il l'application à un risque connu ?* »
