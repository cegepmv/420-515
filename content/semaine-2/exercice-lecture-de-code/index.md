+++
title = "Exercices approfondis — lecture de code"
weight = 2
+++

## 🎯 Objectif

Un exercice distinct pour chacune des quatre stratégies de lecture de code vues dans
[Lire le code des autres]({{< relref "/semaine-2/lecture-de-code" >}}). Chaque partie se termine
par une **vraie petite modification** à apporter à votre projet legacy — rien de comparable en
ampleur au TP1 : ce sont des modifications additives, à faible risque

> 👥 **En équipe.** Travaillez chaque partie sur sa propre branche (ex. `lecture-topdown`,
> `lecture-bottomup`, etc.), avec un commit et une petite Pull Request à la fin — une belle
> occasion de réutiliser vos réflexes de la page [Git 2/2]({{< relref "/semaine-2/git-avance" >}}).
> Ces branches d'exercice n'ont pas besoin d'être conservées après validation en équipe.
>
> 💡 Ces consignes sont volontairement **génériques** : à vous d'adapter chaque étape aux
> ressources, entités et conventions réelles de votre projet. Si un détail technique mentionné
> (ex. génération de code, profils de configuration, mappers) ne s'applique pas à votre projet,
> ignorez-le simplement.

---

## 1️⃣ Lecture top-down — ajouter un endpoint de comptage

**Mise en pratique de la stratégie « depuis les points d'entrée ».**

1. Choisissez un contrôleur simple exposant une liste de ressources dans votre projet.
2. **Formulez une hypothèse** avant de lire le détail : d'après le nom du contrôleur et de la
   méthode, que devrait faire cette méthode ? Repérez ensuite 2-3 **"beacons"** (indices dans le
   code qui confirment votre hypothèse sans qu'il faille tout lire — ex. le nom de la méthode
   appelée sur le service, le type de retour, la présence ou l'absence d'un tri).
3. Repérez la méthode qui répond à la requête `GET` de liste et **descendez** l'appel couche par
   couche : quelle méthode du service appelle-t-elle ? Quelle méthode d'accès aux données
   (repository, DAO, ORM) est finalement invoquée ?
4. Schématisez (sur papier ou dans vos notes d'équipe) la chaîne complète : `Contrôleur → Service →
   Accès aux données`, avec le nom exact de chaque méthode traversée.
5. **Modification à réaliser** : ajoutez un nouvel endpoint qui retourne uniquement le **nombre**
   d'éléments de cette ressource (ex. `{"count": 6}`), en réutilisant la méthode de liste déjà
   comprise plutôt qu'en réinventant une requête. Ajoutez au moins un test qui vérifie que le
   compte retourné correspond au nombre d'éléments existants.

   ⚠️ **Si votre projet génère son code d'API à partir d'une spécification** (ex. contract-first
   avec OpenAPI/Swagger) : les endpoints ne s'ajoutent pas en annotant directement une méthode du
   contrôleur, mais en modifiant d'abord le fichier de spécification, puis en relançant un build
   pour régénérer l'interface/le squelette de code, avant d'implémenter la méthode dans le
   contrôleur. Repérez si c'est le cas de votre projet avant de commencer.

<details>
<summary>❓ Question de vérification</summary>

Si vous deviez ajouter une **validation** sur le corps d'une requête `POST` pour cette même
ressource, à quelle couche l'ajouteriez-vous en priorité (contrôleur, service ou accès aux
données) ? Pourquoi ? Et : un de vos *beacons* attendus était-il absent ou trompeur ? Qu'est-ce que
cela vous a appris ?
</details>

---

## 2️⃣ Lecture bottom-up — ajouter un attribut à une entité

**Mise en pratique de la stratégie « depuis le modèle de données ».**

1. Choisissez une entité simple dans votre projet (peu de champs, peu de logique).
2. Avant de modifier quoi que ce soit, utilisez la recherche d'utilisations de votre IDE (« Find
   Usages ») sur la classe entière : listez **tous** les fichiers qui la référencent (accès aux
   données, objets de transfert/DTO, mapper, contrôleur, tests).

   ⚠️ **Si votre projet contient plusieurs implémentations concurrentes** de l'accès aux données
   pour une même entité (ex. plusieurs technologies de persistance illustrées en parallèle), avant
   d'aller plus loin, repérez dans la configuration du projet **laquelle est réellement active**
   (profil actif, variable d'environnement, indicateur de configuration). Ne perdez pas de temps
   sur les implémentations inactives.
3. **Modification à réaliser** : ajoutez un nouveau champ simple à cette entité (ex. un champ texte
   optionnel). En remontant depuis l'entité, mettez à jour, dans l'ordre : l'implémentation
   d'accès aux données **active** (identifiée à l'étape 2) si une requête explicite liste les
   colonnes ; le schéma ou l'objet de transfert exposé à l'extérieur (DTO) — attention, si votre
   projet **génère** ce DTO automatiquement à partir d'une spécification (ex. OpenAPI), ce n'est
   pas un fichier à modifier directement : il faut modifier la spécification source puis
   régénérer ; puis le mapper (s'il y en a un), qui doit être mis à jour pour transférer le
   nouveau champ entre l'entité et l'objet exposé.
4. Relancez un build si nécessaire pour régénérer le code généré, démarrez l'application et
   vérifiez avec l'interface de test de l'API (Swagger UI ou équivalent) ou `curl` que le nouveau
   champ apparaît dans la réponse.

<details>
<summary>❓ Question de vérification</summary>

Combien de fichiers avez-vous dû modifier au total pour que ce seul nouveau champ soit visible de
bout en bout ? Qu'est-ce que ce nombre vous apprend sur le couplage entre les couches de ce projet ?
</details>

---

## 3️⃣ Tests existants comme documentation vivante

**Mise en pratique de la stratégie « les tests avant le code ».**

1. Choisissez une classe de tests **complète** que vous n'avez pas encore explorée.
2. **Sans regarder l'implémentation**, lisez seulement les noms de méthodes de test et leurs
   assertions. Pour chaque test, écrivez une phrase en français décrivant le comportement attendu
   (ex. « Quand on demande une ressource qui n'existe pas, l'API répond 404 »).
3. Allez ensuite lire la classe de production correspondante et vérifiez si votre hypothèse était
   juste.
4. **Modification à réaliser** : identifiez, avec votre équipe, **un cas limite qui n'est
   actuellement couvert par aucun test** (ex. liste vide, paramètre manquant, valeur limite).
   Écrivez ce nouveau test. S'il échoue, implémentez le changement minimal nécessaire dans le code
   de production pour le faire passer, en respectant le style déjà utilisé dans la classe (s'il
   passe déjà du premier coup, documentez simplement pourquoi dans un commentaire de test).

<details>
<summary>❓ Question de vérification</summary>

Un test qui n'existe pas pour un cas donné veut-il dire que ce cas est forcément buggé ? Pourquoi
faut-il rester prudent face à une suite de tests incomplète ?
</details>

---

## 4️⃣ Suivre l'exécution avec le débogueur

**Mise en pratique de la stratégie « observer plutôt que deviner ».**

1. Choisissez un endpoint qui traverse plusieurs couches avec un peu de logique (récupération d'un
   élément précis par identifiant, ou une recherche avec un critère).
2. Posez un point d'arrêt dans la méthode du contrôleur, démarrez l'application en mode **Debug**
   (🐞), puis déclenchez une vraie requête via l'interface de test de l'API, Postman ou `curl`.
3. Utilisez *Step Into* pour entrer dans le service, puis dans la couche d'accès aux données. À
   chaque étape, notez la valeur réelle d'au moins une variable clé, **et vérifiez le type
   concret** affiché par le débogueur pour l'objet d'accès aux données injecté : si plusieurs
   implémentations existent dans votre projet, seule celle correspondant à la configuration active
   s'exécute réellement.
4. **Modification à réaliser** : à partir de ce que vous avez observé en direct (et non deviné en
   lisant le code), identifiez et implémentez **une petite amélioration justifiée par
   l'observation** — par exemple, exposer dans la réponse un champ déjà présent dans l'entité ou le
   résultat intermédiaire observé dans le débogueur, mais absent de l'objet retourné au client
   (rappel : comme à l'exercice 2, si cet objet est généré automatiquement, il faut d'abord
   modifier la source de génération).
5. Fournissez une capture d'écran du panneau **Variables** du débogueur au moment clé de votre
   observation, en plus du code modifié.

<details>
<summary>❓ Question de vérification</summary>

Y a-t-il eu un moment où l'état réel observé dans le débogueur **différait** de ce que vous aviez
anticipé en lisant seulement le code ? Qu'est-ce que cela vous apprend sur les limites de la
lecture statique seule ?
</details>

---

## ✅ Auto-vérification en équipe

- [ ] Les quatre exercices ont chacun donné lieu à une petite modification livrée (branche + commit).
- [ ] Chaque modification est **additive** et n'a pas touché aux fichiers déjà identifiés comme
      contenant des bogues intentionnels réservés aux semaines suivantes.
- [ ] Les quatre questions de vérification ont été discutées en équipe, pas seulement lues.
- [ ] Chaque équipe a une trace écrite des observations faites pendant chaque exercice
      (schémas de chaîne d'appels, hypothèses de test, captures du débogueur).
