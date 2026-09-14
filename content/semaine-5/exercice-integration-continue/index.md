+++
title = "Exercice : mettre en place un pipeline CI"
weight = 4
draft = true
+++

## 🎯 Objectif

Configurer, exécuter et diagnostiquer un pipeline d'intégration continue réel sur votre propre
dépôt, en réinvestissant les notions vues dans
[Intégration continue]({{% relref "/semaine-5/integration-continue" %}}).

---

## 🧩 Partie 1 — Créer le pipeline

1. En équipe, créez le fichier de configuration adapté à votre plateforme
   (`.github/workflows/ci.yml` pour GitHub Actions, ou `.gitlab-ci.yml` pour GitLab CI) à partir du
   gabarit vu dans les notes de cette semaine.
2. Identifiez les étapes minimales nécessaires : récupération du code, installation du JDK,
   exécution de `./mvnw -B clean verify`.
3. Poussez la configuration sur une branche dédiée et ouvrez une Pull/Merge Request.

---

## 🟢 Partie 2 — Obtenir un premier pipeline vert

1. Observez l'exécution du pipeline sur votre plateforme (onglet **Actions** sur GitHub, ou
   **CI/CD → Pipelines** sur GitLab).
2. Si le pipeline échoue, ouvrez les journaux d'exécution et diagnostiquez la cause avant de
   corriger (erreur de syntaxe YAML, version du JDK incorrecte, fichier non commité, test qui
   échoue réellement).
3. Répétez jusqu'à obtenir un pipeline entièrement vert.

---

## 🔴 Partie 3 — Provoquer volontairement un échec

1. Une fois le pipeline vert, modifiez temporairement un test existant pour qu'il échoue (ex.
   changez une valeur attendue dans une assertion) ou introduisez un petit bogue volontaire dans le
   code.
2. Poussez ce changement et confirmez que le pipeline détecte bien l'échec (❌ visible sur la
   Pull/Merge Request).
3. Annulez la modification (ou corrigez le bogue) et confirmez le retour au vert.

<details>
<summary>❓ Question</summary>

Pourquoi est-il utile de **voir un pipeline échouer** avant de le voir réussir, plutôt que de faire
confiance à un premier essai vert ?

**Réponse** : un pipeline qui n'a jamais été observé en échec pourrait être mal configuré au point
de toujours réussir peu importe l'état réel du code (ex. une étape de test qui ne s'exécute pas
vraiment, ou dont l'échec n'est pas propagé). Provoquer un échec volontaire confirme que le
pipeline **détecte réellement** les problèmes, et pas seulement qu'il s'exécute sans erreur.
</details>

---

## 🔗 Partie 4 — Intégrer la suite de tests de votre équipe

1. Ajoutez au moins un nouveau test (unitaire ou de régression, vus dans l'exercice précédent) au
   dépôt de votre équipe, sur la même branche ou une nouvelle Pull/Merge Request.
2. Confirmez que le pipeline CI exécute automatiquement ce nouveau test, sans configuration
   additionnelle.
3. Fusionnez la Pull/Merge Request une fois le pipeline vert.

---

## ✅ Auto-vérification

- [ ] J'ai un fichier de workflow CI fonctionnel dans mon dépôt.
- [ ] Mon pipeline s'exécute automatiquement à chaque push/Pull Request.
- [ ] J'ai diagnostiqué au moins un échec de pipeline à partir de ses journaux.
- [ ] J'ai confirmé que le pipeline détecte bien un échec provoqué volontairement.
- [ ] Ma suite de tests d'équipe est exécutée automatiquement par le pipeline.
