+++
title = "Exercice : tests unitaires, régression et couverture"
weight = 2
draft = true
+++

## 🎯 Objectif

Mettre en pratique les notions vues dans
[Tests automatisés]({{% relref "/semaine-5/tests-automatises" %}}) : écrire un test unitaire, un
test de régression, mesurer la couverture de code, et corriger un test mal écrit.

---

## 🧩 Partie 1 — Le test de régression désactivé

1. Ouvrez `PetAgeValidatorTest.java` et repérez le test annoté `@Disabled`
   (`shouldReturnFalseWhenBirthDateIsOlderThan50Years`).
2. Retirez l'annotation `@Disabled` et relancez la classe de tests. Confirmez qu'il échoue bien.
3. En équipe, décidez : faut-il corriger le **test** (pour refléter la vraie limite de 100 ans), le
   **code** (le message d'erreur qui ment toujours sur « 50 years »), ou les deux ? Justifiez votre
   choix par écrit (2-3 phrases).
4. Appliquez votre correction, puis relancez la classe de tests jusqu'à ce qu'elle passe au vert.

---

## 📦 Partie 2 — Ajouter un test unitaire

1. Choisissez une méthode de votre projet qui n'a **aucun** test actuellement.
2. Écrivez un test unitaire pour cette méthode, en respectant la structure Arrange-Act-Assert et un
   nommage clair (`should<Résultat>When<Scénario>`).
3. Assurez-vous que votre test échouerait si le comportement de la méthode était cassé — pour vous
   en convaincre, modifiez temporairement la méthode pour introduire un bogue volontaire, confirmez
   que votre test échoue, puis annulez la modification.

---

## 📊 Partie 3 — Couverture de code avec JaCoCo

1. Ajoutez le plugin `jacoco-maven-plugin` à votre `pom.xml` (voir l'extrait de configuration dans
   les notes de cette semaine).
2. Lancez `mvn clean test` et ouvrez `target/site/jacoco/index.html`.
3. Identifiez une classe avec une couverture de lignes faible (< 60 %) et écrivez au moins un
   nouveau test pour l'améliorer.
4. Relancez `mvn clean test` et confirmez que la couverture de cette classe a augmenté.

<details>
<summary>❓ Question</summary>

Après votre nouveau test, la couverture de lignes de la classe est passée de 40 % à 90 %. Est-ce
suffisant pour affirmer que la classe est maintenant bien testée ?

**Réponse** : pas nécessairement — il faut vérifier que le nouveau test contient bien des
assertions pertinentes sur le comportement attendu, et pas seulement qu'il exécute des lignes sans
rien vérifier. La couverture de branches (les deux côtés d'un `if`) mérite aussi d'être inspectée,
pas seulement la couverture de lignes globale.
</details>

---

## 🧹 Partie 4 — Réviser un test mal écrit (revue par les pairs)

1. Formez un binôme d'équipes. Chaque équipe partage avec l'autre un test qu'elle juge « mal écrit »
   dans son propre projet (nom peu clair, plusieurs assertions non reliées, dépendance à un autre
   test, valeurs magiques non expliquées).
2. L'équipe qui reçoit le test propose une réécriture qui applique les bonnes pratiques vues cette
   semaine, puis explique son choix à l'équipe d'origine.
3. Chaque équipe intègre au moins une amélioration suggérée dans son propre projet.

---

## ✅ Auto-vérification

- [ ] J'ai réactivé et corrigé le test de régression désactivé (ou justifié pourquoi le laisser tel quel).
- [ ] J'ai écrit un nouveau test unitaire qui échouerait si le comportement testé était cassé.
- [ ] J'ai configuré JaCoCo et consulté un rapport de couverture réel de mon projet.
- [ ] J'ai amélioré la couverture d'au moins une classe faiblement testée.
- [ ] J'ai révisé un test avec une autre équipe et appliqué au moins une amélioration.
