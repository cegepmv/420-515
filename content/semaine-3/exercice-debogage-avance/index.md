+++
title = "Exercice : outils de débogage"
weight = 2
+++

## 🎯 Objectif

Mettre en pratique, sur votre propre projet, les 4 outils vus dans
[Techniques de débogage (2/2)]({{% relref "/semaine-3/debogage-avance" %}}) : débogueur pas à pas,
breakpoint conditionnel, journalisation stratégique, bissection.

> 👥 **En équipe.** Chaque partie peut être faite par une personne différente, puis partagée en
> équipe — l'objectif est que tout le monde ait manipulé chaque outil au moins une fois.

---

## 🧩 Partie 1 — Débogueur pas à pas

1. Choisissez une méthode de votre projet qui contient une boucle ou une condition non triviale.
2. Posez un breakpoint au début de cette méthode, démarrez en mode Debug, et déclenchez-la
   réellement (appel API, test, ou exécution de l'application).
3. Utilisez *Step Over* pour avancer ligne par ligne, et *Step Into* au moins une fois pour entrer
   dans une méthode appelée.
4. Notez : une valeur de variable qui vous a surpris, et le nom exact de la classe concrète
   affichée par le débogueur si une interface est impliquée.

---

## 🎯 Partie 2 — Breakpoint conditionnel

1. Repérez une collection ou une boucle qui traite plusieurs éléments (ex. une liste retournée par
   un endpoint, ou une boucle `for` sur un tableau).
2. Posez un breakpoint conditionnel qui ne s'arrête que sur un élément précis (par identifiant, par
   une valeur nulle, ou par une valeur hors de l'intervalle attendu).
3. Vérifiez que le programme s'exécute normalement jusqu'à ce cas précis, sans s'arrêter avant.

<details>
<summary>❓ Question</summary>

Pourquoi un breakpoint conditionnel est-il particulièrement utile pour un bogue qui **ne se produit
que sur certaines données**, plutôt qu'un breakpoint classique ?

**Réponse** : un breakpoint classique s'arrête à **chaque** itération, ce qui est trop lent quand
seule une itération précise (parmi des centaines) est en cause. Le breakpoint conditionnel filtre
automatiquement pour vous amener directement au bon endroit.
</details>

---

## 📝 Partie 3 — De `println` à la journalisation

1. Trouvez (ou ajoutez temporairement) un `System.out.println` de débogage dans votre projet.
2. Remplacez-le par un appel à un logger (`logger.debug(...)` ou équivalent dans votre langage),
   avec un message qui inclut le contexte utile (identifiants, valeurs pertinentes).
3. Vérifiez que le message apparaît bien dans la console/les logs au niveau `DEBUG`.

> 💡 Si votre langage/framework n'utilise pas SLF4J (ex. Python, JavaScript), utilisez l'équivalent
> local (`logging` en Python, un logger structuré en Node.js) — le principe des niveaux reste le
> même.

---

## 🔍 Partie 4 — Bissection

**Option A — dans le code** : sur un comportement que vous ne comprenez pas bien, commentez la
moitié d'un bloc suspect, relancez le scénario, et notez si le comportement change. Répétez sur la
moitié restante.

**Option B — dans l'historique (`git bisect`)** : sur votre projet, identifiez un ancien commit où
un comportement précis était différent d'aujourd'hui (ex. un test qui passait avant et échoue
maintenant, ou l'inverse). Utilisez `git bisect start`, `git bisect good`/`bad` pour retrouver le
commit exact où le comportement a changé.

```bash
git bisect start
git bisect bad                 # HEAD (ou un commit récent) présente le problème
git bisect good <ancien-hash>  # ce commit ne présentait pas le problème
# testez à chaque arrêt, puis :
git bisect good   # ou
git bisect bad
# ... jusqu'au commit fautif
git bisect reset
```

---

## 🐛 Partie 5 — Chasse aux bogues sur le projet du cours

Comme dans un vrai emploi, le code hérité que vous maintenez cette session n'est pas parfait —
certains comportements ne correspondent pas à ce qui est documenté ou attendu. Voici quelques
**tickets de bogue**, dans le format que vous recevriez en entreprise. C'est l'occasion de mettre en
pratique les 4 outils ci-dessus sur du **vrai code**, pas un exemple simplifié.

Choisissez **au moins 2** des tickets suivants, et pour chacun : (1) reproduisez le comportement
décrit, (2) utilisez l'outil suggéré pour confirmer votre hypothèse sur la cause exacte, (3)
proposez un correctif, (4) vérifiez manuellement que le comportement est maintenant celui attendu.

---

**🎫 Ticket #1 — Les visites d'un animal ne s'affichent pas dans le bon ordre**
- **Endpoint concerné** : `GET /api/pets/{petId}` (ou tout endpoint qui retourne un `Pet` avec ses
  visites), pour un animal ayant **plusieurs visites à des dates différentes**.
- **Comportement attendu** : la visite la **plus récente** apparaît en premier dans la liste
  `visits`.
- **Comportement observé** : la visite la plus **ancienne** apparaît en premier.
- **Outil suggéré** : Débogueur pas à pas — posez un breakpoint sur la méthode qui construit/trie
  cette liste, et inspectez son contenu juste avant de la retourner.

**🎫 Ticket #2 — Un animal existant retourne une erreur 404 selon son identifiant**
- **Endpoint concerné** : `GET /api/owners/{ownerId}/pets/{petId}`.
- **Comportement attendu** : retourne `200 OK` avec les détails de l'animal, pour n'importe quel
  `petId` valide appartenant à cet `ownerId` (confirmé par ailleurs via `GET /api/pets/{petId}`
  qui, lui, retourne bien l'animal).
- **Comportement observé** : retourne `404 Not Found` pour certains `petId` valides — le problème
  semble lié à la **valeur numérique** de l'identifiant plutôt qu'à l'animal lui-même.
- **Repro** : créez suffisamment de nouveaux animaux (via `POST`) pour qu'un `petId` dépasse une
  centaine, puis interrogez cet animal via l'endpoint ci-dessus.
- **Outil suggéré** : Breakpoint conditionnel — arrêtez-vous uniquement quand la comparaison entre
  deux identifiants retourne un résultat inattendu.

**🎫 Ticket #3 — Le message d'erreur sur l'âge d'un animal ne correspond pas à ce qui est réellement accepté**
- **Endpoint concerné** : `POST /api/owners/{ownerId}/pets` (ou `PUT .../pets/{petId}`) avec un
  `birthDate` ancien.
- **Comportement attendu (d'après le message d'erreur affiché)** : une date de naissance de plus de
  **50 ans** devrait être rejetée (`400 Bad Request`, message *"Birth date cannot be older than 50
  years"*).
- **Comportement observé** : une date de naissance de, par exemple, 70 ans est **acceptée** sans
  erreur. Seule une date de plus de 100 ans est rejetée.
- **Outil suggéré** : Débogueur pas à pas — posez un breakpoint dans la classe de validation
  concernée et inspectez la valeur limite réellement comparée.

**🎫 Ticket #4 — Un nom vide est refusé pour un propriétaire, mais accepté pour un animal**
- **Endpoints concernés** : `POST /api/owners` (création d'un propriétaire) et
  `PUT /api/pets/{petId}` (mise à jour d'un animal).
- **Comportement attendu** : la même règle métier (« un nom ne doit pas être vide ») s'applique de
  façon cohérente, peu importe l'entité.
- **Comportement observé** : envoyer un `firstName` (ou `lastName`) égal à une **chaîne vide
  `""`** à la création d'un propriétaire est rejeté (`400 Bad Request`). Envoyer un `name` égal à
  une chaîne vide **`""`** à la mise à jour d'un animal est accepté sans erreur (`204 No
  Content`).
- **Outil suggéré** : Journalisation — ajoutez un `logger.debug` dans chacune des deux méthodes de
  validation concernées (une dans le contrôleur des propriétaires, une dans celui des animaux)
  pour comparer précisément ce qu'elles acceptent/rejettent.

**🎫 Ticket #5 — La recherche de propriétaires par nom de famille échoue avec une apostrophe**
- **Endpoint concerné** : `GET /api/owners?lastName=...`.
- **Pré-requis pour reproduire** : ce ticket concerne l'implémentation utilisée quand l'application
  tourne avec le profil de dépôt `jpa` (voir `application.properties` :
  `spring.profiles.active=h2,jpa` au lieu de la configuration par défaut). Démarrez l'application
  avec ce profil avant de tester.
- **Comportement attendu** : retourne la liste des propriétaires dont le nom de famille commence
  par la valeur fournie (ou une liste vide s'il n'y en a aucun).
- **Comportement observé** : avec une valeur contenant une apostrophe (ex. `O'Brien`), la requête
  échoue avec une erreur serveur (`500`) au lieu de retourner un résultat.
- **Outil suggéré** : Débogueur pas à pas — arrêtez-vous juste avant l'exécution de la requête et
  inspectez la chaîne de requête réellement construite avec cette valeur.

---

Si un comportement vous semble avoir changé **récemment** sans que vous compreniez pourquoi (plutôt
qu'être un problème présent depuis toujours), essayez plutôt la **bissection** (`git bisect`) sur
l'historique du projet pour identifier le commit responsable.

> 💡 Une fois un problème confirmé et corrigé, ouvrez une Pull Request avec une courte explication
> de la cause — exactement la démarche attendue en entreprise pour ce genre de correctif.

---

## ✅ Auto-vérification

- [ ] J'ai utilisé *Step Over* et *Step Into* au moins une fois chacun sur du vrai code.
- [ ] Un breakpoint conditionnel m'a permis d'atteindre directement un cas précis dans une
      collection, sans m'arrêter à chaque itération.
- [ ] J'ai remplacé au moins un `println` par un appel de logger avec le bon niveau.
- [ ] J'ai réduit un espace de recherche par bissection (code ou `git bisect`) jusqu'à isoler une
      seule cause.
- [ ] J'ai trouvé et corrigé au moins 2 problèmes réels du projet du cours en confirmant chaque
      hypothèse avec l'outil de débogage approprié.

> 💡 Gardez vos notes de cette pratique — elles pourront être utiles pour un travail pratique à
> venir.
