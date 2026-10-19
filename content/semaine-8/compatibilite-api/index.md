+++
title = "Compatibilité d'API : changements compatibles et changements cassants"
weight = 3
draft = false
+++

Cette page s'appuie sur deux sources : les **API Improvement Proposals (AIP)** de Google —
guides de conception d'API publics et largement cités dans l'industrie — et le code réel du
projet utilisé dans ce cours.

---

## 1️⃣ Une API est un contrat

> « Les API sont fondamentalement des contrats avec les utilisateurs, et les utilisateurs écrivent
> souvent du code contre une API qui est ensuite déployé en production avec l'attente qu'il
> continue de fonctionner. »
> — [Google AIP-180, Backwards compatibility](https://google.aip.dev/180) (traduction libre)

Concrètement : du code écrit aujourd'hui contre votre API doit continuer à fonctionner demain,
même si vous publiez de nouvelles versions — à moins d'avoir explicitement averti vos utilisateurs
du contraire (via SemVer, voir la page précédente).

---

## 2️⃣ Trois types de compatibilité (AIP-180)

AIP-180 distingue trois formes de compatibilité à préserver :

| Type | Définition (AIP-180) |
|---|---|
| **Source** | Du code écrit contre une version précédente doit continuer à **compiler** contre une nouvelle version |
| **Wire** (protocole réseau) | Du code écrit contre une version précédente doit pouvoir **communiquer correctement** avec un serveur plus récent (les formats de sérialisation/désérialisation restent compatibles) |
| **Sémantique** | Du code écrit contre une version précédente doit continuer à recevoir **ce à quoi un développeur raisonnable s'attendrait** |

> ⚠️ AIP-180 précise aussi : « il n'est pas toujours évident de savoir si un changement est
> compatible ou non [...] ce guide doit être traité comme indicatif, plutôt que comme une liste
> exhaustive de tous les changements possibles. » Les règles qui suivent sont donc des repères
> solides, pas une liste parfaitement complète.

### Illustrer les trois types avec un seul endpoint

Les trois types de compatibilité ne se chevauchent pas : un changement peut casser **un seul**
des trois, sans toucher aux deux autres. Prenons `GET /owners/{id}`, qui retourne aujourd'hui :

```json
{ "id": 1, "firstName": "George", "lastName": "Franklin" }
```

| Type cassé | Changement illustratif | Pourquoi ce type précisément |
|---|---|---|
| **Source** seulement | Retirer un paramètre d'une méthode Java générée par le client SDK (ex. `OwnerClient.getOwner(int id)` devient `getOwner(int id, boolean includePets)`) | Le code client existant ne **compile plus** (paramètre manquant), même si le JSON échangé sur le réseau n'a pas changé |
| **Wire** seulement | Changer le type JSON du champ `id` de nombre (`1`) à chaîne (`"1"`) | Le code client **compile** toujours (il garde la même signature), mais la **désérialisation** échoue à l'exécution — le format transmis sur le réseau n'est plus celui attendu |
| **Sémantique** seulement | Sans changer ni structure ni type, faire en sorte que `lastName` retourne désormais le nom de jeune fille plutôt que le nom d'usage | Le code client compile, le JSON a la même forme et les mêmes types — mais la **valeur reçue ne correspond plus** à ce qu'un développeur attend raisonnablement d'un champ nommé `lastName` |

> 💡 En pratique, un seul changement casse souvent **plusieurs** types à la fois (ex. retirer un
> champ casse à la fois le wire et, si du code dépendait de ce champ, le source). Les distinguer
> aide surtout à **diagnostiquer** pourquoi un changement qui « semble mineur » (même structure,
> même type) peut quand même casser des clients — comme dans l'exemple sémantique ci-dessus.

---

## 3️⃣ Ajouter des éléments : généralement compatible, avec des exceptions

Selon AIP-180, section *Adding components* :

> « En général, de nouveaux composants (interfaces, méthodes, messages, champs, énumérations) PEUVENT
> être ajoutés à des API existantes dans la même version majeure. »

Mais la même section pose des conditions précises :

- « De **nouveaux champs obligatoires** (*required*) NE DOIVENT PAS être ajoutés à des messages de
  requête ou des ressources existants. »
- « Tout champ pouvant être rempli par le client DOIT avoir un comportement par défaut qui
  correspond au comportement d'avant l'introduction du champ. »
- « Tout champ précédemment rempli par le serveur DOIT continuer à l'être. »

Et selon la section *Removing or renaming components* :

> « Des composants existants NE DOIVENT PAS être retirés d'une API existante dans la même version
> majeure. Retirer un composant est un changement non rétrocompatible. [...] Renommer un composant
> est sémantiquement équivalent à « retirer puis ajouter ». »

| Type de changement | Compatible selon AIP-180 ? |
|---|---|
| Ajouter un champ **optionnel** à une réponse | ✅ Oui |
| Ajouter un champ **obligatoire** à une requête existante | ❌ Non — casse les clients qui ne l'envoient pas |
| Ajouter une nouvelle route/endpoint | ✅ Oui |
| Retirer un champ existant | ❌ Non, explicitement cassant |
| Renommer un champ existant | ❌ Non — « équivalent à retirer puis ajouter » |
| Changer le type d'un champ existant | ❌ Non (section *Changing the type of fields* : « ne doivent pas avoir leur type changé, même si le nouveau type est compatible au niveau du protocole ») |

<details>
<summary>🤔 Testez-vous</summary>

Une équipe ajoute un champ `email` optionnel à la réponse JSON d'un endpoint `GET /owners/{id}`
existant. Les clients actuels ignorent simplement ce champ inconnu. Selon AIP-180, ce changement
est-il compatible, et quel type d'incrément SemVer justifie-t-il ?

**Réponse** : oui, compatible — un champ ajouté à une réponse, que les anciens clients peuvent
ignorer sans erreur, respecte la règle « nouveaux composants PEUVENT être ajoutés ». Selon la
règle 7 de SemVer (page précédente), l'ajout d'une fonctionnalité rétrocompatible justifie un
incrément **MINEUR** (ex. `4.0.2` → `4.1.0`), pas un incrément MAJEUR.
</details>

---

## 4️⃣ Exemples réels de l'industrie : GitHub et Stripe

Les règles d'AIP-180 ne sont pas propres à Google : d'autres fournisseurs d'API largement utilisés,
dont GitHub lui-même, publient des règles quasi identiques — et documentent publiquement de vrais
changements classés « cassant » ou « non cassant ».

### GitHub REST API : une liste officielle quasi identique à AIP-180

La documentation officielle de l'API REST de GitHub définit, presque terme à terme, les mêmes deux
catégories que celles vues à la section 3️⃣ :

> « Les changements cassants (*breaking changes*) sont des changements qui peuvent potentiellement
> briser une intégration. [...] Les changements cassants comprennent :
> - Retirer une opération entière
> - Retirer ou renommer un paramètre
> - Retirer ou renommer un champ de réponse
> - Ajouter un nouveau paramètre obligatoire
> - Rendre obligatoire un paramètre auparavant optionnel
> - Changer le type d'un paramètre ou d'un champ de réponse
> - Retirer des valeurs d'énumération
> - Changer les exigences d'authentification ou d'autorisation »
> — [GitHub Docs, *API Versions*](https://docs.github.com/en/rest/about-the-rest-api/api-versions)

> « Tout changement additif (non cassant) sera disponible dans toutes les versions d'API
> supportées. Les changements additifs comprennent :
> - Ajouter une opération
> - Ajouter un paramètre optionnel
> - Ajouter un champ de réponse
> - Ajouter des valeurs d'énumération »
> — [GitHub Docs, *API Versions*](https://docs.github.com/en/rest/about-the-rest-api/api-versions)

GitHub va même plus loin qu'une simple classification : chaque nouvelle version de son API REST
(nommée par date, ex. `2022-11-28`) reste **supportée au moins 24 mois** après la sortie d'une
version plus récente, et le client choisit sa version via un en-tête dédié —
`X-GitHub-Api-Version: 2022-11-28` — exactement le même mécanisme de versionnage par en-tête que
Stripe (`Stripe-Version`, voir section 6️⃣).

### Stripe : deux vrais changements tirés du changelog public

Stripe publie un [changelog public](https://docs.stripe.com/changelog) de chaque changement
apporté à son API, classé explicitement **Breaking** ou **Non-breaking**. En voici deux exemples
réels, documentés tels quels par Stripe :

**Un vrai changement cassant** — renommer un paramètre *et* changer son type :

> « Renomme le paramètre `filters.countries` en `filters.country` et change son type d'un tableau
> de chaînes à une seule chaîne [...]. Le paramètre `filters.countries` n'existe plus. Si votre
> code transmet un tableau de codes de pays à `filters.countries`, vous devez le remplacer par
> `filters.country` avec une seule valeur de chaîne. Transmettre l'ancien nom de paramètre ou une
> valeur de tableau entraîne une erreur. »
> — [Stripe Changelog, *Renames the countries filter to country*](https://docs.stripe.com/changelog/endive/2026-09-30/rename-financial-connections-countries-filter-to-country)

Ce changement illustre **à la fois** deux règles vues à la section 3️⃣ : renommer un champ
(« équivalent à retirer puis ajouter ») et changer son type — deux changements que AIP-180 qualifie
chacun, indépendamment, de non rétrocompatibles.

**Un vrai changement additif (classé non cassant)** — ajouter des valeurs d'énumération :

> « La propriété `account_numbers.status` [...] prend maintenant en charge deux nouvelles valeurs :
> `pending` et `expired`. [...] Si votre intégration utilise `account_numbers.status`, mettez-la à
> jour pour gérer ces nouvelles valeurs en plus de `transactable` et `deactivated`. »
> — [Stripe Changelog, *Adds pending and expired statuses*](https://docs.stripe.com/changelog/endive/2026-09-30/pending-and-expired-status-added-to-financial-connections-account-numbers)

> ⚠️ Ce deuxième exemple illustre bien la nuance soulevée par AIP-180 à la section 2️⃣ : Stripe
> classe cet ajout comme **non cassant** (il respecte la règle « on PEUT ajouter de nouvelles
> valeurs d'énumération »), tout en précisant lui-même qu'une intégration cliente qui ne gère pas
> explicitement une valeur inconnue de ce champ devrait être mise à jour. Un changement peut donc
> être « compatible selon les règles » tout en exigeant, en pratique, une vigilance du côté client.

---

## 5️⃣ Exemple réel du projet : deux versions d'un même endpoint

Le projet utilisé dans ce cours expose **réellement** deux versions de la liste des propriétaires,
déclarées dans son fichier `openapi.yml` :

```yaml
# openapi.yml — extrait réel du projet (src/main/resources/openapi.yml)
/owners:
  post:
    tags: [owners]
    operationId: addOwner
    summary: Adds a pet owner
  get:
    tags: [owners]
    operationId: listOwners
    summary: Lists pet owners
    description: Returns an array of pet owners.
    # ... retourne un tableau JSON brut (type: array)

/v2/owners:
  get:
    tags: [owner-v2]
    operationId: listOwnersPage
    summary: Lists pet owners with pagination
    description: Returns a page of pet owners.
    # ... retourne un objet paginé (page), pas un tableau brut
```

Deux contrôleurs distincts implémentent ces deux routes dans le code :

```java
// OwnerRestControllerV1.java — GET /api/owners
public ResponseEntity<List<OwnerDto>> listOwners(String lastName) {
    Collection<Owner> owners = (lastName != null)
        ? this.clinicService.findOwnerByLastName(lastName)
        : this.clinicService.findAllOwners();
    return new ResponseEntity<>(ownerMapper.toOwnerDtoCollection(owners), HttpStatus.OK);
}
```

```java
// OwnerRestControllerV2.java — GET /api/v2/owners
public ResponseEntity<OwnerPageDto> listOwnersPage(String lastName, Integer page, Integer size) {
    Page<Owner> owners = this.clinicService.findOwners(
        lastName, PageRequest.of(page, size, Sort.by("id")));
    return new ResponseEntity<>(ownerMapper.toOwnerPageDto(owners), HttpStatus.OK);
}
```

Le chemin d'URI détermine donc **quelle méthode Java** traite la requête : `listOwners` (V1)
retourne directement un `List<OwnerDto>` sérialisé en tableau JSON, tandis que `listOwnersPage`
(V2) retourne un `OwnerPageDto` (objet paginé avec métadonnées). La forme de la réponse (tableau
brut vs objet paginé) est **différente** — un client qui attend un tableau JSON planterait en
recevant un objet. C'est précisément le genre de changement que la section *Removing or renaming
components* d'AIP-180 qualifie d'incompatible s'il avait été appliqué **directement** sur
`/owners` : la forme de la réponse n'est pas la même, donc d'anciens clients cesseraient de
fonctionner.

La solution appliquée dans ce projet correspond à la recommandation d'AIP-181
(*Stability levels*, section *Major versions*) :

> « Quand des changements cassants deviennent nécessaires, le producteur de l'API DEVRAIT créer la
> prochaine version majeure de l'API [...]. »
> — [Google AIP-181](https://google.aip.dev/181)

`/owners` (V1) continue de fonctionner exactement comme avant, sans aucune modification, pendant
que `/v2/owners` (V2) introduit le nouveau format pour les clients prêts à l'adopter.

---

## 6️⃣ Comment signaler une version au client : chemin d'URI vs en-tête HTTP

Le projet du cours utilise le **chemin d'URI** (`/v2/owners`) pour distinguer les versions —
l'approche la plus visible et la plus simple à documenter. Ce n'est toutefois pas la seule
pratique répandue dans l'industrie. Stripe, par exemple, utilise un **en-tête HTTP dédié** :

> « Les requêtes faites avec curl utilisent la version d'API par défaut de votre compte Stripe
> [...] à moins que vous ne la remplaciez en définissant l'en-tête `Stripe-Version`. »
> — [Stripe, Versioning](https://docs.stripe.com/api/versioning)

| Stratégie | Exemple concret | Avantage |
|---|---|---|
| **Chemin d'URI** | `/v2/owners` (ce projet) | Visible directement dans l'URL; simple à documenter et à tester avec `curl` |
| **En-tête HTTP** | `Stripe-Version: 2024-09-30` | L'URL reste identique entre versions; le client choisit sa version sans changer d'adresse |

> 💡 Peu importe la stratégie choisie, le principe reste le même : **la version précédente
> continue de fonctionner sans changement**, le temps que les clients migrent vers la nouvelle.

---

## 📚 Références

- Google — [AIP-180: Backwards compatibility](https://google.aip.dev/180).
- Google — [AIP-181: Stability levels](https://google.aip.dev/181).
- Stripe — [API Versioning](https://docs.stripe.com/api/versioning).
- GitHub Docs — [API Versions (REST)](https://docs.github.com/en/rest/about-the-rest-api/api-versions).
- Stripe Changelog — [Renames the countries filter to country](https://docs.stripe.com/changelog/endive/2026-09-30/rename-financial-connections-countries-filter-to-country).
- Stripe Changelog — [Adds pending and expired statuses to Financial Connections account numbers](https://docs.stripe.com/changelog/endive/2026-09-30/pending-and-expired-status-added-to-financial-connections-account-numbers).
- Code source du projet du cours — `openapi.yml`, `OwnerRestControllerV1.java`, `OwnerRestControllerV2.java`.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice SemVer et compatibilité]({{% relref "/semaine-8/exercice-semver-compatibilite" %}})
pour appliquer ces critères à des changements concrets.
{{% /notice %}}
