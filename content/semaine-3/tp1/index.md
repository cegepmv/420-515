+++
title = "TP1 — Recherche de personnes"
weight = 5
draft = true
+++

> Inspiré du laboratoire *Epic-1* du dépôt `philipmvcegep/simple_website`
> (https://github.com/philipmvcegep/simple_website/wiki/Epic-1), adapté pour réinvestir les
> notions de **git** et de **lecture de code** (semaines 1-2), ainsi que **Maven** et les
> **techniques de débogage** vues à la semaine 3 du cours 420-515.

## Mise en contexte

Vous recevez un **frontend React déjà fonctionnel** (dossier `frontend/`) qui affiche une liste de
personnes et permet de les filtrer avec une barre de recherche. Ce frontend a été écrit par
quelqu'un d'autre : votre premier travail n'est pas d'écrire du code, mais de **lire et comprendre
du code que vous n'avez pas écrit** — exactement la compétence pratiquée dans
[lecture de code]({{% relref "/semaine-2/lecture-de-code" %}}) (semaine 2). Ensuite, vous allez
construire de A à Z le backend Spring Boot qui remplacera le mécanisme actuel (base de données
SQLite chargée dans le navigateur) par une vraie API REST.

> ⚠️ **Important** : le dépôt qui vous est remis contient **uniquement** le dossier `frontend/`.
> Il n'y a **aucun dossier `backend/` fourni** — ni scaffold, ni squelette, ni `pom.xml` de départ.
> Vous devez créer le backend **de A à Z** vous-mêmes, à partir de
> [start.spring.io](https://start.spring.io/).

## Équipes

Équipes de 2 à 3 personnes.

## Livrables et pondération (10 %)

| Critère | Pondération |
|---|---|
| Le frontend affiche et filtre des données provenant de **votre** API Spring Boot (et non plus de sql.js) | 27 % |
| Historique Git : commits atomiques, branches `feature/*`, au moins une Pull Request avec revue croisée entre équipes | 27 % |
| Réponses écrites aux questions de lecture de code (section « Étape 0 ») | 12 % |
| Maven : `pom.xml` enrichi et documenté (Étape 2bis) | 12 % |
| Débogage : bogue diagnostiqué et corrigé à l'aide des techniques de la semaine 3, démarche documentée (Étape 3bis) | 17 % |
| Swagger/OpenAPI généré et fonctionnel, et persistance réelle des écritures dans `people.csv` (Étape 3ter) | 5 % |
| **Bonus** — Tests automatisés sur au moins un endpoint (capture d'écran des tests passants à joindre à la PR) | + 5 % |

> Les tests automatisés (`@WebMvcTest`/`@SpringBootTest`) ne sont **pas encore vus au moment de ce
> TP** — ils sont donc **facultatifs et bonifiés** (jusqu'à +5 %), pas une exigence de base.

Remise : PR ouverte sur `main` (via `release/*`) + capture d'écran des tests qui passent (si vous
en avez fait), avant la date indiquée sur le plan de cours.

---

## Étape 0 — Lire le frontend avant d'y toucher

Avant d'écrire une seule ligne de backend, explorez `frontend/` en équipe et répondez par écrit
(dans un fichier `NOTES-LECTURE.md` à la racine de votre dépôt) aux questions suivantes. Utilisez
les stratégies vues en semaine 2 :

1. **Stratégie descendante (top-down)** : ouvrez `src/App.jsx`. Quel est le point d'entrée visuel
   (le composant racine) ? Quelle fonction est appelée au chargement de la page (`useEffect`) et
   que fait-elle ?
2. **Stratégie ascendante (bottom-up)** : ouvrez `src/db.js` et `src/utils/search.js`
   indépendamment, **avant** de relire `App.jsx`. Qu'est-ce que `loadDatabaseFromCSV` construit
   exactement ? Qu'est-ce que `searchPeople` retourne ? Assemblez ensuite mentalement le tout —
   est-ce cohérent avec votre lecture descendante ?
3. **Repères (« beacons », Brooks 1983)** : quels mots-clés dans le code vous ont permis de
   deviner rapidement qu'il s'agissait d'une recherche par préfixe (`LIKE '...%'`) plutôt qu'une
   recherche par sous-chaîne (`LIKE '%...%'`) ? Citez la ligne exacte.
4. **Données** : d'où viennent les données affichées (`public/data/people.csv`) ? Combien de
   colonnes, combien de lignes ?
5. **Point d'accroche pour le futur backend** : la fonction `searchPeople` dans
   `src/utils/search.js` est celle que vous devrez **remplacer**. Sans encore coder, écrivez en
   français ce que devra faire son équivalent côté backend (méthode HTTP, chemin, paramètre de
   requête, forme de la réponse JSON attendue).

## Étape 1 — Mettre en place le dépôt et le flux Git

Ce TP suit **Git Flow** (comme dans l'exercice
[git avancé]({{% relref "/semaine-2/exercice-git-avance" %}}), avec une branche `release/*`) :

- `main` : toujours déployable, ne contient que du code fusionné depuis `release/*`. C'est **la
  seule branche corrigée par l'enseignant** — ce qui s'y trouve doit être final.
- `release/*` : branche de stabilisation créée à partir de `develop` lorsque les fonctionnalités
  visées sont prêtes ; sert à finaliser (tests, corrections mineures) avant la fusion vers `main`.
- `develop` : intégration des fonctionnalités, ne contient que du code fusionné depuis des
  branches `feature/*`.
- `feature/<nom-court>` : une branche par fonctionnalité ci-dessous, créée à partir de `develop`.

Chaque fonctionnalité doit être livrée par une **Pull Request** de `feature/*` vers `develop`,
relue par l'autre équipe qui vous sera assignée (revue croisée, comme dans l'exercice git avancé).
Une fois `develop` prêt, ouvrez une Pull Request `develop` → `release/*`, puis, après validation,
une dernière Pull Request `release/*` → `main`.

> 🔒 **Push direct interdit sur `main`, `release/*` et `develop`** : le dépôt est protégé par une
> *ruleset* GitHub qui **bloque tout push direct** sur ces trois branches — tout changement doit
> obligatoirement passer par une Pull Request (avec au moins une revue approuvée avant fusion). Si
> votre push est rejeté, c'est normal : créez une branche `feature/*` (ou `release/*` selon
> l'étape) et ouvrez une Pull Request. Ceci n'est pas une contrainte technique arbitraire — c'est
> le même mécanisme utilisé en entreprise pour éviter qu'un changement non relu ou non finalisé
> casse la branche principale.

## Étape 2 — Créer le backend Spring Boot

1. Générez un projet Spring Boot via [start.spring.io](https://start.spring.io/) (dépendances :
   `Spring Web`, `Validation`) dans un dossier `backend/` à la racine du dépôt.
2. Créez un enregistrement (`record` ou classe) `Person` avec les champs du CSV : `id`,
   `firstName`, `lastName`, `email`, `gender`.
3. Chargez les données de `frontend/public/data/people.csv` en mémoire au démarrage de
   l'application (liste en mémoire, pas de base de données pour ce TP — gardez ça simple, comme le
   prescrit KISS).

> 📌 Cette liste en mémoire n'est qu'un point de départ : à l'Étape 3ter, vous devrez **aussi
> réécrire le fichier CSV** à chaque `POST`/`PUT`/`DELETE`, pour que les données persistent
> réellement comme si vous aviez une vraie base de données.

## Étape 2bis — Explorer et enrichir le `pom.xml`

Avant d'implémenter les endpoints, prenez cinq minutes pour ouvrir le `pom.xml` généré par
start.spring.io et répondez par écrit (ajoutez ces réponses à `NOTES-LECTURE.md`) :

1. Combien de `<dependency>` avez-vous, et à quoi sert chacune (regardez le `artifactId`) ?
2. Lancez `mvn dependency:tree` à la racine de `backend/`. Repérez une dépendance **transitive**
   (introduite indirectement par `spring-boot-starter-web` ou `spring-boot-starter-validation`) et
   notez laquelle.
3. **Ajoutez une dépendance** utile pour lire le CSV proprement (ex.
   `org.apache.commons:commons-csv`, ou toute librairie équivalente de votre choix) avec le bon
   `scope`, puis relancez `mvn dependency:tree` pour confirmer qu'elle est bien intégrée.
4. Notez, en une phrase, ce que fait la commande `mvn clean package` que ne fait pas `mvn package`
   seul (revoir la phase `clean` du cycle de vie Maven vue en semaine 3 dans les notes de cours
   [Maven (1/2)]({{% relref "/semaine-3/maven" %}})).

## Étape 3 — Endpoints REST

Implémentez, un par un, chacun dans sa propre branche `feature/*` :

| Méthode | Endpoint | Comportement |
|---|---|---|
| `GET` | `/api/persons` | retourne toutes les personnes |
| `GET` | `/api/persons/search?q=...` | retourne les personnes dont le prénom, nom, email **ou** genre commence par `q` (reproduit `searchPeople`) |
| `POST` | `/api/persons` | ajoute une personne (corps JSON) |
| `PUT` | `/api/persons/{id}` | modifie une personne existante |
| `DELETE` | `/api/persons/{id}` | supprime une personne |

**Bonus (facultatif)** : la notion de test automatisé n'est pas encore vue au moment de ce TP,
donc aucun test n'est exigé. Si vous voulez tenter l'exercice, un test (`@WebMvcTest` ou
`@SpringBootTest`) qui vérifie le cas de succès d'au moins un endpoint, avec capture d'écran des
tests passants jointe à votre PR finale, rapporte jusqu'à +5 % (voir la section « Livrables et
pondération »).

> ⚠️ **Point d'attention** : `searchPeople` (voir Étape 0, question 5) compare tout en minuscules
> (`lower(...)`) des deux côtés, et matche un **préfixe** (`LIKE '...%'`), pas une sous-chaîne
> quelconque. En traduisant cette logique en Java, il est facile d'introduire par erreur une
> recherche sensible à la casse, ou une recherche par sous-chaîne au lieu d'un préfixe. Vous en
> aurez besoin à l'étape suivante.

## Étape 3ter — Documenter l'API avec Swagger et rendre les écritures persistantes

1. Ajoutez la dépendance `springdoc-openapi-starter-webmvc-ui` (Swagger) à votre `pom.xml`, puis
   vérifiez que `/swagger-ui/index.html` et `/v3/api-docs` répondent correctement une fois
   l'application démarrée.
2. Ajoutez des annotations `@Tag`/`@Operation` (ou équivalent) sur votre contrôleur pour que chaque
   endpoint soit clairement documenté dans Swagger (méthode, paramètres, réponses possibles).
3. **Persistance réelle** : jusqu'ici, `POST`/`PUT`/`DELETE` ne modifient que la liste en mémoire —
   si vous redémarrez l'application, vos changements disparaissent. À partir de maintenant,
   **chaque appel `POST`/`PUT`/`DELETE`** (que ce soit depuis le frontend **ou** depuis
   l'interface Swagger) doit aussi **réécrire le fichier `people.csv`** sur disque, pour que les
   données survivent à un redémarrage — exactement comme le ferait une vraie base de données.
   Testez ceci en équipe : faites un appel `POST` ou `DELETE` via Swagger, redémarrez
   l'application, et confirmez avec `GET /api/persons` que le changement est toujours là.

## Étape 3bis — Diagnostiquer un comportement inattendu

Une fois `/api/persons/search` implémenté, testez-le manuellement (ou en équipe croisée, testez
celui d'une autre équipe) avec des requêtes qui mélangent les majuscules (ex. `q=MAR`) et des
requêtes qui ne devraient **pas** matcher un résultat au milieu d'un mot (ex. `q=aria` ne devrait
rien retourner pour « Maria », puisque la recherche est par préfixe).

- S'il y a un écart entre le comportement observé et le comportement attendu (défini à l'Étape 0,
  question 5), **utilisez le débogueur pas à pas** (breakpoint dans la méthode du contrôleur ou du
  service concerné, inspection des variables) pour localiser précisément où le comportement
  diverge — ne devinez pas, observez. Voir les notes de cours
  [Techniques de débogage (2/2)]({{% relref "/semaine-3/debogage-avance" %}}).
- Si aucun écart n'apparaît spontanément, **introduisez-en un intentionnellement** (ex. remplacez
  temporairement une comparaison insensible à la casse par une comparaison sensible à la casse),
  puis pratiquez la même démarche de diagnostic pour le retrouver et le corriger.
- Remplacez tout `System.out.println` de débogage temporaire par un **logger SLF4J**
  (`LoggerFactory.getLogger(...)`) au niveau `debug`, comme vu en semaine 3 — aucun `println` ne
  doit rester dans le code remis.
- **Les logs doivent aussi être écrits dans un fichier**, pas seulement dans la console :
  configurez `logging.file.name` (ex. `logging.file.name=logs/backend.log`) dans
  `application.properties`. Le dossier `logs/` doit être ignoré par Git (ajoutez-le à
  `.gitignore`) — ce n'est pas un artefact à committer, mais vous devez pouvoir montrer le fichier
  généré (ex. capture d'écran ou extrait) comme preuve de votre démarche de débogage.

**Livrable** : un court paragraphe dans un fichier `NOTES-DEBOGAGE.md` à la racine du dépôt,
décrivant (1) le comportement observé vs attendu, (2) l'hypothèse de départ, (3) l'outil utilisé
pour confirmer (débogueur et/ou logs) et (4) la correction apportée. Une capture d'écran de la
session de débogage (variables inspectées, ou extrait de logs) est un plus mais n'est pas
obligatoire.

## Étape 4 — Reconnecter le frontend

Modifiez `frontend/src/App.jsx` (et supprimez `src/db.js`, `src/utils/search.js` qui deviennent
inutiles) pour que la recherche appelle `GET http://localhost:8080/api/persons/search?q=...` via
`fetch` au lieu d'interroger une base sql.js locale. Le tableau affiché doit rester identique du
point de vue de l'utilisateur.

## Étape 5 (optionnelle, bonus) — CRUD dans l'interface

Ajoutez un formulaire simple pour créer/modifier/supprimer une personne depuis l'interface,
branché sur les endpoints `POST`/`PUT`/`DELETE`.

---

## Rappel : ce n'est pas un projet à partir de rien

Le frontend fourni est un point de départ **volontairement imparfait et non documenté** — c'est
voulu : en entreprise, vous hériterez souvent de code sans documentation. L'étape 0 (lecture de
code) n'est pas une formalité, elle sera évaluée.

> 📌 Le dépôt de départ (frontend uniquement) vous sera fourni séparément par l'enseignant·e, qui
> précisera aussi la date de remise exacte.
