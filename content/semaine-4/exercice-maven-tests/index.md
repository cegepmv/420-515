+++
title = "Exercice : exécution des tests avec Maven"
weight = 2
+++

## 🎯 Objectif

Manipuler concrètement l'exécution des tests avec Maven, sur votre propre projet, en réinvestissant
les notions vues dans
[Maven (2/2)]({{% relref "/semaine-4/maven-tests" %}}).

---

## 🧩 Partie 1 — Exécuter et cibler des tests

1. Lancez `mvn test` sur votre projet et repérez, dans la console, le résumé final
   (`Tests run: X, Failures: Y, Errors: Z, Skipped: W`).
2. Choisissez une classe de test existante et relancez-la seule avec `-Dtest=NomDeLaClasse`.
3. Ciblez ensuite une seule méthode de cette classe avec `-Dtest=NomDeLaClasse#nomDeLaMéthode`.

---

## 📄 Partie 2 — Lire un rapport Surefire

1. Ouvrez le dossier `target/surefire-reports/` généré après votre `mvn test`.
2. Choisissez un fichier `.txt` et identifiez : le nombre de tests exécutés, leur temps d'exécution
   total, et si des tests ont été ignorés (`Skipped`).
3. Si un test échoue, ouvrez le rapport correspondant et repérez la trace de la première assertion
   qui a échoué.

---

## 🚫 Partie 3 — `-DskipTests` vs `-Dmaven.test.skip`

1. Lancez `mvn package -DskipTests` : confirmez dans la console que les classes de test sont
   **compilées**, mais qu'aucun test ne s'exécute.
2. Lancez `mvn package -Dmaven.test.skip=true` : confirmez cette fois que la compilation des tests
   elle-même est **sautée**.
3. Modifiez temporairement une classe de test pour qu'elle ne compile plus (ex. retirez un `;`),
   puis relancez les deux commandes précédentes. Laquelle des deux révèle le problème de
   compilation, et laquelle le masque complètement ? Annulez ensuite votre modification.

<details>
<summary>❓ Question</summary>

Un script de déploiement automatisé utilise `-Dmaven.test.skip=true` pour accélérer les livraisons
fréquentes. Quel risque cela introduit-il, comparé à `-DskipTests` ?

**Réponse** : si le code de test cesse de compiler (ex. après un renommage dans le code de
production), `-Dmaven.test.skip=true` masque complètement le problème — le build continue de
réussir. `-DskipTests`, lui, compile quand même les tests : une erreur de compilation dans les
tests serait détectée immédiatement, même sans exécuter leurs assertions.
</details>

---

## ✅ Auto-vérification

- [ ] J'ai exécuté une classe de test précise avec `-Dtest`.
- [ ] J'ai lu un rapport Surefire et identifié le nombre de tests exécutés/ignorés.
- [ ] Je peux expliquer la différence entre `-DskipTests` et `-Dmaven.test.skip=true`.
- [ ] J'ai identifié, dans mon propre projet, si des classes de test suivent bien la convention de
      nommage reconnue par Surefire (`*Test`, `*Tests`, `*TestCase`).
