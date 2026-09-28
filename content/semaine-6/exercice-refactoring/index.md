+++
title = "Exercice : identifier et appliquer des réusinages"
weight = 2
draft = false
+++

## 🎯 Objectif

Pratiquer, sur votre propre projet, la démarche complète vue dans
[Réusinage (refactoring) et code smells]({{% relref "/semaine-6/refactoring" %}}) : repérer un
*code smell* réel, le corriger avec un réusinage catalogué, par petites étapes vérifiées par votre
suite de tests (semaine 5) à chaque fois.

---

## 🧩 Partie 1 — Chasser les *code smells*

1. En équipe, parcourez un module de votre projet (ou de l'application legacy fournie) et repérez
   **au moins trois** *code smells* différents parmi ceux du catalogue de la semaine (méthode trop
   longue, duplication, classe trop grosse, liste de paramètres trop longue, obsession des
   primitifs, *feature envy*, *data clumps*, chaînes de conditions répétées).
2. Pour chacun, notez : le nom du smell, le fichier et la méthode concernés, et **une phrase**
   expliquant pourquoi ce smell rend le code plus difficile à comprendre ou à modifier.
3. Ne corrigez rien encore — l'objectif de cette partie est seulement d'apprendre à **reconnaître**
   un smell avant de foncer dans la correction.

<details>
<summary>🤔 Testez-vous</summary>

Vous trouvez une méthode de 80 lignes qui fonctionne parfaitement et que personne n'a eu besoin de
modifier depuis deux ans. Est-ce un problème à corriger en priorité ?

**Réponse** : pas nécessairement. Un *code smell* signale un risque **potentiel** pour la
maintenabilité future, pas un défaut de fonctionnement actuel. Si ce code n'a besoin d'aucune
évolution et reste stable, le bénéfice de le réusiner est faible — le risque (même limité par des
tests) ne se justifie que si ce code doit bientôt être modifié ou étendu. Priorisez plutôt les
smells situés dans du code que vous savez devoir toucher bientôt.
</details>

---

## 🟢 Partie 2 — Confirmer le filet de sécurité avant de commencer

1. Choisissez **un seul** smell repéré à la Partie 1, situé dans du code déjà couvert par au moins
   un test (semaine 5). Si aucun test ne couvre ce code, écrivez d'abord un test de
   caractérisation qui capture son comportement actuel — **avant** de continuer.
2. Lancez la suite de tests complète et confirmez qu'elle est entièrement verte. Notez le nombre de
   tests exécutés (ex. « 42 tests, 0 échec ») — ce sera votre point de référence.

---

## 🔁 Partie 3 — Réusiner par petites étapes

1. Appliquez **un seul** réusinage catalogué à la fois (ex. *Extract Method*, *Rename*, *Extract
   Variable*), de préférence via la commande dédiée de votre IDE plutôt que manuellement.
2. Relancez immédiatement la suite de tests.
   - **Verte** : committez cette étape avec un message clair et précis (ex. « Extract
     calculateTotal() from buildInvoiceSummary() »), puis passez à l'étape suivante.
   - **Rouge** : annulez cette transformation avant d'aller plus loin — ne continuez jamais sur un
     état rouge.
3. Répétez jusqu'à ce que le smell choisi soit corrigé. Vous devriez terminer avec **plusieurs
   petits commits**, pas un seul commit massif.

<details>
<summary>🤔 Testez-vous</summary>

Après avoir appliqué trois réusinages d'affilée sans relancer les tests entre chacun, la suite de
tests échoue. Que faites-vous ?

**Réponse** : puisque les trois transformations n'ont pas été vérifiées individuellement, il est
impossible de savoir directement laquelle a causé l'échec. La bonne pratique est de revenir en
arrière (`git checkout`/`git reset` vers le dernier état vert) et de réappliquer les
transformations **une à la fois**, en relançant les tests après chacune — exactement la discipline
que cette partie de l'exercice cherche à installer.
</details>

---

## 🔍 Partie 4 — Confirmer l'absence de changement de comportement

1. Comparez le nombre de tests exécutés et leur résultat à la fin de la Partie 3 avec ceux notés à
   la Partie 2 — ils doivent être **identiques** (même nombre de tests, tous verts).
2. Si vous avez repéré un bogue réel pendant le réusinage (ex. un cas non couvert par les tests
   existants), **ne le corrigez pas dans les mêmes commits** : terminez d'abord le réusinage
   (tests verts, commits de réusinage seuls), puis ouvrez un commit séparé, clairement identifié
   comme une correction, pour le bogue.
3. Poussez vos commits sur une branche dédiée et ouvrez une Pull Request décrivant, pour chaque
   commit, quel réusinage catalogué a été appliqué et pourquoi.

---

## ✅ Auto-vérification

- [ ] J'ai identifié au moins trois *code smells* réels dans mon projet, avec justification.
- [ ] Le code réusiné était déjà couvert par un test avant que je ne le modifie (ou je l'ai
      couvert d'abord).
- [ ] J'ai appliqué mes réusinages un à la fois, en relançant les tests après chaque étape.
- [ ] Mon historique Git montre plusieurs petits commits de réusinage, pas un seul commit massif.
- [ ] Le nombre de tests et leurs résultats sont identiques avant et après mes réusinages.
- [ ] Si j'ai trouvé un bogue en cours de route, je l'ai corrigé dans un commit séparé, pas mélangé
      au réusinage.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[activité pratique]({{% relref "/semaine-6/activite-pratique" %}}) pour
réinvestir cette démarche en équipe sur un module plus substantiel de l'application legacy.
{{% /notice %}}
