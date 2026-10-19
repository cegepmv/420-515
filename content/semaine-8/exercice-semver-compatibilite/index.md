+++
title = "Exercice : SemVer et compatibilité d'API"
weight = 4
draft = false
+++

## 🎯 Objectif

Appliquer les règles vues dans [SemVer]({{% relref "/semaine-8/semver" %}}) et
[Compatibilité d'API]({{% relref "/semaine-8/compatibilite-api" %}}) à des changements concrets.

---

## 🧩 Partie 1 — Classer des changements selon SemVer

Pour chacun des changements suivants, déterminez s'il justifie un incrément **MAJEUR**, **MINEUR**
ou de **CORRECTIF**, en citant la règle de la spécification SemVer qui justifie votre réponse
(règles 6, 7 ou 8 — voir [SemVer]({{% relref "/semaine-8/semver" %}})) :

1. Un endpoint `GET /owners/{id}` retournait une erreur HTTP 500 sur un propriétaire sans animal;
   le correctif retourne maintenant correctement la liste vide, sans changer la forme de la
   réponse dans les autres cas.
2. Un nouveau champ optionnel `email` est ajouté à la réponse JSON de `GET /owners/{id}`.
3. Le champ `telephone` est retiré de la réponse JSON de tous les endpoints `owners`.
4. Un nouvel endpoint `GET /owners/{id}/visits` est ajouté, sans toucher à aucun endpoint existant.
5. Le type du champ `id` passe de `integer` à `string` dans toutes les réponses JSON.

---

## 🧩 Partie 2 — Analyser l'exemple réel du projet

En consultant `src/main/resources/openapi.yml` et les classes `OwnerRestControllerV1` /
`OwnerRestControllerV2` du projet :

1. Identifiez la différence exacte de forme de réponse entre `GET /owners` (V1) et
   `GET /v2/owners` (V2).
2. En vous basant sur AIP-180 (section *Removing or renaming components*, voir
   [Compatibilité d'API]({{% relref "/semaine-8/compatibilite-api" %}})), expliquez pourquoi cette
   différence n'aurait **pas** pu être appliquée directement à `GET /owners` sans casser les
   clients existants.
3. Expliquez, en une phrase, pourquoi la stratégie choisie (créer `/v2/owners` plutôt que modifier
   `/owners`) correspond à la recommandation d'AIP-181 citée dans les notes de cette semaine.

---

## 🧩 Partie 3 — Concevoir un changement compatible

Le modèle `Owner` du projet possède actuellement les champs `address`, `city`, `telephone` (voir
`model/Owner.java`). Concevez l'ajout d'un nouveau champ optionnel à ce modèle (ex. `email`) en
répondant aux questions suivantes, chacune directement tirée des critères d'AIP-180 vus dans les
notes :

1. Ce nouveau champ doit-il être **obligatoire** ou **optionnel** dans les requêtes de création
   (`POST /owners`) pour rester compatible avec les clients existants ? Justifiez avec la règle
   d'AIP-180 citée dans les notes.
2. Si un client existant n'envoie jamais ce champ, quel comportement par défaut doit adopter le
   serveur, selon la même règle ?
3. Quel incrément SemVer ce changement justifie-t-il (MAJEUR, MINEUR, ou CORRECTIF) ?

<details>
<summary>❓ Corrigé — Partie 1</summary>

1. **CORRECTIF** — règle 6 : correction rétrocompatible d'un comportement incorrect, sans
   changement de l'API publique.
2. **MINEUR** — règle 7 : ajout d'une fonctionnalité rétrocompatible (un champ optionnel ignoré
   sans erreur par les anciens clients).
3. **MAJEUR** — règle 8 : retirer un champ est un changement non rétrocompatible (voir aussi
   AIP-180, section *Removing or renaming components*).
4. **MINEUR** — règle 7 : un nouvel endpoint n'affecte aucun client existant.
5. **MAJEUR** — règle 8, et explicitement couvert par AIP-180 (*Changing the type of fields* :
   « ne doivent pas avoir leur type changé »).
</details>

---

## ✅ Auto-vérification

- [ ] Je peux justifier chaque classement MAJEUR/MINEUR/CORRECTIF en citant la règle SemVer
      exacte qui s'applique.
- [ ] Je peux expliquer, avec mes mots, la différence entre les trois types de compatibilité
      définis par AIP-180 (source, wire, sémantique).
- [ ] Je sais expliquer pourquoi le projet du cours utilise `/v2/owners` plutôt que de modifier
      `/owners` directement.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[activité pratique]({{% relref "/semaine-8/activite-pratique" %}}) pour faire
évoluer réellement une API sans casser sa compatibilité.
{{% /notice %}}
