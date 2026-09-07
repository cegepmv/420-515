+++
title = "Exercice : manipuler pom.xml"
weight = 4
+++

## 🎯 Objectif

Manipuler concrètement les notions vues dans
[Maven (1/2)]({{% relref "/semaine-3/maven" %}}) : ajouter une dépendance, lire un arbre de
dépendances, et comprendre le cycle de vie via de vraies commandes `mvn`.

---

## 🧩 Partie 1 — Explorer l'arbre de dépendances existant

1. À la racine de votre projet, lancez :
   ```bash
   mvn dependency:tree
   ```
2. Repérez une dépendance **transitive** (une ligne indentée qui n'apparaît PAS directement dans
   votre `pom.xml`) et notez : quelle dépendance directe l'a introduite, et sa version.
3. Si la sortie affiche `(omitted for conflict)` quelque part, notez les deux versions en jeu et
   laquelle a été retenue.

---

## 📦 Partie 2 — Ajouter une dépendance

1. Choisissez une petite librairie utilitaire pertinente pour votre projet (ex. une librairie de
   validation, de manipulation de dates, ou de génération de données de test).
2. Ajoutez-la dans la section `<dependencies>` de votre `pom.xml`, avec le bon `scope` (`compile`
   si utilisée dans le code de production, `test` si utilisée seulement dans les tests).
3. Relancez `mvn dependency:tree` et confirmez qu'elle apparaît bien, avec ses éventuelles propres
   dépendances transitives.

> 💡 Cherchez les coordonnées exactes (`groupId`/`artifactId`/`version`) sur
> [Maven Central](https://search.maven.org/).

---

## 🧪 Partie 3 — Cycle de vie et rapport de tests

1. Lancez `mvn clean test`.
2. Repérez dans la console le résumé du plugin Surefire (`Tests run: X, Failures: Y, Errors: Z`).
3. Ouvrez le dossier `target/surefire-reports/` et examinez un rapport `.txt` généré pour une
   classe de test.

<details>
<summary>❓ Question</summary>

Si vous lancez `mvn package` directement (sans lancer `mvn test` avant), les tests seront-ils quand
même exécutés ?

**Réponse** : oui — `package` est une phase **postérieure** à `test` dans le cycle de vie Maven, et
exécuter une phase déclenche automatiquement toutes celles qui la précèdent. Les tests seront donc
exécutés avant l'empaquetage, sans commande séparée.
</details>

---

## ✅ Auto-vérification

- [ ] J'ai identifié au moins une dépendance transitive et sa dépendance directe d'origine.
- [ ] J'ai ajouté une nouvelle dépendance avec le bon `scope`.
- [ ] J'ai localisé et lu un rapport Surefire généré par `mvn test`.
- [ ] Je peux expliquer pourquoi `mvn package` exécute aussi `test` sans commande explicite.
