+++
title = "Exercice : écrire vos premiers tests JUnit"
weight = 4
+++

## 🎯 Objectif

Écrire une première classe de test JUnit 6 (Jupiter) sur votre application legacy, en réinvestissant les
notions vues dans [Introduction aux tests automatisés (JUnit)]({{% relref "/semaine-4/junit" %}}).

---

## 🧩 Partie 1 — Premier test, sans mock

1. Choisissez une classe de votre projet dont la logique ne dépend **pas** de Spring, d'une base de
   données ou d'un appel réseau (un objet du domaine avec une méthode qui calcule ou vérifie
   quelque chose, comme `Owner.getPet(...)` dans le projet du cours).
2. Créez la classe de test correspondante au bon endroit (`src/test/java/...`, même package que la
   classe testée, nom se terminant par `Test` ou `Tests`).
3. Écrivez au moins deux méthodes `@Test` : un cas où le résultat attendu est présent/vrai, et un
   cas où il est absent/faux (ex. un élément trouvé vs un élément non trouvé).

---

## 🔁 Partie 2 — Factoriser avec `@BeforeEach`

1. Si vos deux tests répètent la même préparation d'objets, extrayez-la dans une méthode annotée
   `@BeforeEach`.
2. Confirmez que vos deux tests passent toujours après cette factorisation.
3. Ajoutez un troisième test qui réutilise le même état préparé par `@BeforeEach`, sans dupliquer sa
   construction.

---

## 🧪 Partie 3 — Exception, assertions groupées et test paramétré

1. Écrivez un test avec `assertThrows` qui vérifie qu'une méthode de votre projet lance bien une
   exception dans un cas invalide (ex. argument `null` ou hors limites).
2. Regroupez au moins deux assertions liées au même scénario avec `assertAll`, puis provoquez
   volontairement l'échec des **deux** en même temps pour confirmer qu'elles sont bien rapportées
   ensemble dans un seul résultat de test.
3. Transformez un test qui vérifie plusieurs valeurs d'entrée similaires (ou écrivez-en un nouveau)
   en test paramétré avec `@ParameterizedTest` et `@ValueSource` ou `@CsvSource`.

---

## 🎭 Partie 4 — Un premier mock

1. Repérez, dans le projet, un exemple existant de `mock(...)` (ex. `PetAgeValidatorTest`) ou de
   `@MockitoBean` (ex. `OwnerRestControllerV1Tests`).
2. En équipe, expliquez à voix haute pourquoi ce mock est nécessaire ici : quelle dépendance
   remplace-t-il, et pourquoi serait-il difficile ou peu souhaitable d'utiliser la vraie dépendance
   dans ce test ?
3. Sans nécessairement l'exécuter, identifiez une méthode de votre projet qui aurait besoin d'un
   mock similaire pour être testée en isolation (ex. une méthode qui appelle un service externe ou
   une base de données).

---

## 🧨 Partie 5 — Faire échouer un test intentionnellement

1. Modifiez temporairement une assertion pour qu'elle échoue à coup sûr (ex. changez la valeur
   attendue).
2. Lancez le test et lisez attentivement le message d'échec produit par JUnit (« expected: ...
   but was: ... »).
3. Annulez votre modification et confirmez que le test repasse au vert.

<details>
<summary>❓ Question</summary>

Pourquoi est-il utile, en apprenant à écrire des tests, de voir volontairement un test échouer au
moins une fois avant de le voir passer ?

**Réponse** : cela confirme que le test vérifie réellement quelque chose — un test qui ne peut
jamais échouer (assertion toujours vraie, mauvaise valeur comparée) ne protège contre rien, même
s'il s'affiche en vert. Observer un vrai échec, puis un vrai passage, prouve que le test est
sensible au comportement qu'il est censé vérifier.
</details>

---

## ✅ Auto-vérification

- [ ] J'ai écrit une classe de test JUnit avec au moins deux méthodes `@Test`.
- [ ] Mes tests sont placés dans `src/test/java/...`, dans le même package que la classe testée.
- [ ] J'ai utilisé `@BeforeEach` pour factoriser une préparation commune.
- [ ] J'ai écrit un test avec `assertThrows` et un autre avec `assertAll`.
- [ ] J'ai transformé ou écrit un test paramétré (`@ParameterizedTest`).
- [ ] J'ai expliqué pourquoi un mock existant du projet est nécessaire à cet endroit précis.
- [ ] J'ai observé un échec de test intentionnel et lu son message d'erreur avant de le corriger.
