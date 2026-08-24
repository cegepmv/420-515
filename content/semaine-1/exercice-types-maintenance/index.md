+++
title = "Exercice : étiqueter les types de maintenance"
weight = 7
+++

## 🎯 Objectif

Manipuler concrètement la classification ISO/IEC 14764 (corrective, adaptative, perfective,
préventive) en l'appliquant à des cas réalistes, **avant** de la mémoriser par cœur.

## 📝 Déroulement

Pour chacun des 6 scénarios ci-dessous, déterminez le type de maintenance dominant
(**corrective**, **adaptative**, **perfective** ou **préventive**) et essayez de formuler
vous-même, en une phrase, pourquoi. Cliquez ensuite sur chaque encadré pour vérifier votre réponse.

---

**Scénario 1** — Un client signale que le total d'une facture est mal arrondi lorsqu'il y a 3
articles ou plus dans le panier.

<details>
<summary>✅ Voir la réponse</summary>

**Corrective.** Le comportement observé (montant mal calculé) ne correspond pas au comportement
attendu — c'est un bogue existant qu'on répare.
</details>

---

**Scénario 2** — Le fournisseur de paiement en ligne annonce la fin de support de son ancienne API
dans 6 mois; l'équipe doit migrer son code d'intégration vers la nouvelle version avant cette date.

<details>
<summary>✅ Voir la réponse</summary>

**Adaptative.** Rien ne change pour l'utilisateur final, mais le logiciel doit s'adapter à un
changement de son environnement externe (une dépendance tierce qui évolue).
</details>

---

**Scénario 3** — Le module d'authentification n'a aucun test automatisé. Aucun bogue n'a été
rapporté, mais l'équipe ajoute des tests pour réduire le risque de régression future.

<details>
<summary>✅ Voir la réponse</summary>

**Préventive.** On modifie le logiciel *avant* qu'un problème ne survienne, pour réduire un risque
futur — rien n'est cassé aujourd'hui.
</details>

---

**Scénario 4** — Une méthode de calcul de remise de 300 lignes fonctionne correctement, mais elle
est presque impossible à modifier. L'équipe la divise en plusieurs méthodes plus petites, sans
changer le résultat pour l'utilisateur.

<details>
<summary>✅ Voir la réponse</summary>

**Perfective.** Rien n'est cassé, mais on améliore la structure interne du code (lisibilité,
facilité de modification future).
</details>

---

**Scénario 5** — Depuis la sortie de la dernière version d'un système d'exploitation mobile,
l'application plante au démarrage sur certains téléphones, car une API système utilisée a changé.

<details>
<summary>✅ Voir la réponse</summary>

**Adaptative.** Le déclencheur est un changement de l'environnement externe (nouvelle version de
l'OS), même si le symptôme ressemble à un bogue.
</details>

---

**Scénario 6** — Un formulaire d'inscription accepte un courriel mal formé sans afficher d'erreur,
ce qui cause des échecs silencieux plus tard dans le processus. On ajoute la validation manquante.

<details>
<summary>✅ Voir la réponse</summary>

**Corrective.** Un défaut du logiciel (validation manquante) est réparé — le comportement attendu
n'était pas respecté.
</details>

---

## 🧮 Auto-évaluation

Une fois les 6 scénarios complétés, vérifiez que :

- [ ] Vous avez identifié les 4 types dans l'ensemble des scénarios.
- [ ] Vos justifications étaient cohérentes avec le déclencheur du changement (bogue existant vs
      changement externe vs amélioration demandée vs anticipation d'un risque).

<details>
<summary>💬 Piège classique à surveiller</summary>

Beaucoup d'étudiants confondent **perfective** et **préventive**. Rappel : si le changement est
demandé parce qu'un **utilisateur ou le marché** veut quelque chose de mieux (nouvelle fonction,
meilleure performance, code plus lisible pour l'équipe), c'est **perfective**. Si le changement est
fait pour **éviter un problème futur** qui n'existe pas encore (ex. : ajouter des tests, refactoriser
un module fragile avant qu'il ne cause un bogue), c'est **préventive**.
</details>
