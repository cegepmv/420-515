+++
title = "Introduction aux tests automatisés (JUnit)"
weight = 3
+++

Jusqu'ici, vérifier qu'un changement de code fonctionne s'est fait **manuellement** : relancer
l'application, cliquer, observer. Cette approche ne passe pas à l'échelle et s'oublie facilement.
Un **test automatisé** encode cette vérification une fois pour toutes, sous forme de code exécutable
en quelques secondes, autant de fois que nécessaire.

---

## 0️⃣ Vocabulaire de base

Avant d'écrire du code, clarifions quelques termes utilisés partout dans ce chapitre (et dans
n'importe quelle documentation ou outil de test) :

- **Assertion** : une vérification unique, du type « je m'attends à ce que cette valeur soit égale
  à celle-là ». En JUnit, ce sont les méthodes comme `assertEquals(...)` ou `assertTrue(...)`. Une
  assertion qui échoue fait échouer le test dans lequel elle se trouve.
- **Cas de test** (*test case*) : une méthode qui vérifie **un seul comportement précis** à l'aide
  d'une ou plusieurs assertions (ex. « trouver un animal par son nom, peu importe la casse »). En
  JUnit, un cas de test est une méthode annotée `@Test`. C'est la plus petite unité qu'on exécute et
  qu'on rapporte individuellement (réussi, échoué, en erreur ou ignoré).
- **Classe de test** : regroupe plusieurs cas de test liés à une même classe ou un même
  comportement du programme (ex. `OwnerTest` regroupe tous les cas de test pour la classe `Owner`).
- **Suite de test** (*test suite*) : un regroupement plus large de classes de test, exécutées
  ensemble — par exemple, « tous les tests unitaires du module `validation` » ou « tous les tests du
  projet ». Quand `mvn test` affiche `Tests run: 47`, ce nombre additionne les cas de test de
  **toutes** les classes exécutées, c'est-à-dire de la suite complète.
- **Résultat d'un cas de test** : chaque cas de test se termine dans l'un de ces quatre états :
  - ✅ **Succès** (*passed*) : toutes les assertions sont vraies.
  - ❌ **Échec** (*failure*) : au moins une assertion a échoué (le comportement attendu n'est pas
    au rendez-vous).
  - 💥 **Erreur** (*error*) : le code a levé une exception imprévue (souvent un bug, ou un test mal
    écrit) — différent d'un échec d'assertion.
  - ⏭️ **Ignoré** (*skipped*) : le cas de test n'a pas été exécuté, par exemple parce qu'il est
    annoté `@Disabled`.
- **Test unitaire** : un test qui vérifie une petite unité de code (souvent une méthode ou une
  classe) de façon isolée, sans dépendances externes réelles (base de données, réseau, fichiers).
  C'est le type de test que vous allez écrire dans ce chapitre. On l'oppose au **test
  d'intégration**, qui vérifie que plusieurs composants fonctionnent correctement ensemble.

Avec ce vocabulaire en tête, la ligne `Tests run: 47, Failures: 0, Errors: 0, Skipped: 1` que vous
verrez plus loin se lit simplement : « la suite complète contient 47 cas de test, aucun échec,
aucune erreur, 1 ignoré ».

---

## 1️⃣ Anatomie d'une classe de test JUnit 6

Le projet utilise déjà **JUnit 6** (la génération actuelle du framework, souvent encore appelée
« Jupiter » pour son moteur de test) — rien à ajouter au `pom.xml`, c'est inclus par
`spring-boot-starter-test`. On peut le confirmer directement avec `mvn dependency:tree` :

```
[INFO]    \- org.junit.jupiter:junit-jupiter:jar:6.0.3:test
[INFO]       +- org.junit.jupiter:junit-jupiter-api:jar:6.0.3:test
[INFO]       +- org.junit.jupiter:junit-jupiter-params:jar:6.0.3:test
[INFO]       \- org.junit.jupiter:junit-jupiter-engine:jar:6.0.3:test
```

> 💡 **Pourquoi 6 et pas 5 ?** Le projet JUnit a abandonné le suffixe « 5 » dans son nom (le dépôt
> s'appelle maintenant simplement `junit-framework`), mais garde les mêmes artefacts Maven
> (`org.junit.jupiter:junit-jupiter-*`) et le même modèle de programmation. La version 6 exige une
> base Java plus récente et modernise certaines dépendances internes, mais l'API que vous utilisez
> au quotidien (`@Test`, `assertEquals`, `@BeforeEach`, etc.) reste la même que celle popularisée
> sous le nom « JUnit 5 ». D'où le nom encore très répandu « Jupiter » pour désigner ce modèle de
> test, peu importe le numéro de version exact du moteur qui l'exécute.

### Structurer un test avec le patron Arrange-Act-Assert (AAA)

Avant de regarder le premier exemple, remarquez sa structure interne : presque tout cas de test
suit le même patron en 3 étapes, souvent abrégé **AAA** :

1. **Arrange** (préparer) : construire les objets nécessaires et mettre en place l'état de départ
   (ex. créer un `Owner`, lui ajouter un `Pet`).
2. **Act** (agir) : exécuter **une seule fois** l'action précise qu'on veut tester — l'appel de
   méthode dont on vérifie le comportement (ex. `owner.getPet("LÉO")`).
3. **Assert** (vérifier) : comparer le résultat obtenu au résultat attendu, à l'aide d'une ou
   plusieurs assertions.

Séparer visiblement ces trois blocs (souvent avec un commentaire ou une ligne vide, comme dans
l'exemple ci-dessous) rend un test **plus facile à lire d'un coup d'œil** : on sait immédiatement
quelle partie prépare les données, laquelle déclenche le comportement testé, et laquelle vérifie
le résultat — même dans un test plus long. C'est aussi un bon signal d'alarme : si l'étape *Act*
contient plusieurs appels différents, ou si des assertions apparaissent avant l'action, c'est
souvent le signe qu'un test essaie de vérifier trop de choses à la fois et devrait être scindé en
plusieurs cas de test plus ciblés.

Voici une première classe de test, volontairement simple, écrite pour la classe `Owner` du projet
(aucun mock, aucun contexte Spring nécessaire — un pur objet Java) :

```java
package org.springframework.samples.petclinic.model;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class OwnerTest {

    @Test
    void shouldFindPetByNameIgnoringCase() {
        // Arrange
        Owner owner = new Owner();
        Pet pet = new Pet();
        pet.setName("Léo");
        owner.addPet(pet);

        // Act
        Pet found = owner.getPet("LÉO");

        // Assert
        assertNotNull(found);
        assertEquals("Léo", found.getName());
    }

    @Test
    void shouldReturnNullWhenPetNameDoesNotExist() {
        Owner owner = new Owner();

        Pet found = owner.getPet("Inconnu");

        assertNull(found);
    }
}
```

| Élément | Rôle |
|---|---|
| `@Test` | Marque une méthode comme un test que JUnit doit exécuter |
| `assertEquals(attendu, réel)` | Échoue si les deux valeurs ne sont pas égales |
| `assertNotNull(valeur)` / `assertNull(valeur)` | Échoue si la valeur est (ou n'est pas) `null` |
| `assertTrue(condition)` / `assertFalse(condition)` | Échoue si la condition est fausse (ou vraie) |
| `import static ...Assertions.*` | Permet d'écrire `assertEquals(...)` plutôt que `Assertions.assertEquals(...)` |

> 💡 Remarquez `shouldFindPetByNameIgnoringCase` : on cherche `"LÉO"` (majuscules) alors que le nom
> enregistré est `"Léo"`. Ce test vérifie précisément le comportement de
> `Owner.getPet(String, boolean)` : la comparaison se fait `toLowerCase()` des deux côtés, donc la
> casse ne devrait pas compter. Un bon test choisit ses données pour vérifier un comportement
> précis, pas seulement « un cas qui marche ».

### Autres assertions utiles

```java
@Test
void petListShouldContainExactlyTwoElements() {
    Owner owner = new Owner();
    owner.addPet(new Pet());
    owner.addPet(new Pet());

    assertEquals(2, owner.getPets().size());
}
```

<details>
<summary>🤔 Testez-vous</summary>

Pourquoi `assertEquals("Léo", found.getName())` place-t-il la valeur **attendue** en premier
argument, et la valeur **réelle** (obtenue par le code testé) en second ?

**Réponse** : c'est la convention JUnit — l'ordre des arguments détermine le message d'échec
affiché (« expected: \<Léo\> but was: \<...\> »). Inverser les deux ne change pas si le test
passe ou échoue, mais rend le message d'erreur trompeur en cas d'échec, ce qui ralentit le
diagnostic.
</details>

---

## 2️⃣ Regrouper plusieurs assertions avec `assertAll`

Un piège fréquent avec plusieurs `assertEquals` d'affilée : dès que le **premier** échoue, JUnit
arrête immédiatement le test — les assertions suivantes ne s'exécutent jamais, même si elles
auraient révélé d'autres problèmes. `assertAll` regroupe plusieurs vérifications liées au **même**
scénario et les exécute **toutes**, même si certaines échouent, pour un diagnostic complet en un
seul passage :

```java
@Test
void shouldCreatePetWithExpectedFields() {
    Owner owner = new Owner();
    Pet pet = new Pet();
    pet.setName("Léo");
    owner.addPet(pet);

    Pet found = owner.getPet("Léo");

    assertAll("propriétés du Pet retrouvé",
        () -> assertNotNull(found),
        () -> assertEquals("Léo", found.getName()),
        () -> assertEquals(owner, found.getOwner())
    );
}
```

> 💡 Sans `assertAll`, si `assertEquals("Léo", found.getName())` échouait à cause d'un bogue, vous
> ne sauriez **jamais**, dans le même essai, si `found.getOwner()` était lui aussi incorrect — il
> faudrait corriger le premier problème, relancer le test, et découvrir le second seulement à ce
> moment-là. `assertAll` rapporte les deux échecs d'un coup.

---

## 3️⃣ Tester qu'une exception est bien lancée

`assertThrows` vérifie qu'un bloc de code lance bien le type d'exception attendu — et retourne
cette exception, pour en inspecter le contenu (ex. le message) :

```java
@Test
void shouldThrowWhenPetNameIsNull() {
    Owner owner = new Owner();

    NullPointerException exception = assertThrows(
        NullPointerException.class,
        () -> owner.getPet(null)
    );

    assertNotNull(exception.getMessage());
}
```

> ⚠️ Un test comme celui-ci révèle un vrai choix de conception à documenter : `Owner.getPet(String,
> boolean)` appelle `name.toLowerCase()` sans jamais vérifier que `name` n'est pas `null` — passer
> `null` provoque donc une `NullPointerException` **non intentionnelle**, plutôt qu'une erreur
> métier claire. Écrire ce test rend ce comportement explicite, que l'équipe décide ensuite de le
> garder tel quel ou de le corriger (ex. lancer une `IllegalArgumentException` avec un message
> clair à la place).

<details>
<summary>🤔 Testez-vous</summary>

Un test utilise `assertThrows(RuntimeException.class, () -> methode())`, alors que `methode()` lance
en réalité une `NullPointerException`. Le test passe-t-il ?

**Réponse** : oui — `NullPointerException` **hérite** de `RuntimeException`, et `assertThrows`
accepte toute sous-classe du type demandé. C'est voulu, mais ça veut aussi dire qu'un type trop
générique (`RuntimeException`, voire `Exception`) peut faire passer un test même si l'exception
réellement lancée n'est pas celle que vous pensiez tester — soyez aussi précis que possible dans le
type attendu.
</details>

---

## 4️⃣ Tests paramétrés : éviter de copier-coller un test pour chaque valeur

Sans outil dédié, tester plusieurs valeurs d'entrée oblige à dupliquer le même test avec seulement
une valeur qui change — ce qui viole la bonne pratique « éviter la duplication » vue ailleurs dans
le cours. `@ParameterizedTest` exécute la **même** méthode de test une fois par valeur fournie :

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

@ParameterizedTest
@ValueSource(strings = {"Léo", "LÉO", "léo", "lÉo"})
void shouldFindPetRegardlessOfCase(String recherche) {
    Owner owner = new Owner();
    Pet pet = new Pet();
    pet.setName("Léo");
    owner.addPet(pet);

    assertNotNull(owner.getPet(recherche));
}
```

Pour des cas plus riches (plusieurs valeurs liées entre elles), `@CsvSource` fournit chaque
combinaison directement :

```java
import org.junit.jupiter.params.provider.CsvSource;

@ParameterizedTest
@CsvSource({
    "Léo, léo, true",
    "Léo, Milo, false"
})
void shouldMatchPetNameCaseInsensitively(String nomEnregistré, String recherche, boolean trouvé) {
    Owner owner = new Owner();
    Pet pet = new Pet();
    pet.setName(nomEnregistré);
    owner.addPet(pet);

    assertEquals(trouvé, owner.getPet(recherche) != null);
}
```

> 💡 Ce n'est pas juste une question de concision : un test paramétré rend visible, d'un seul coup
> d'œil dans le rapport Surefire, **quelle combinaison précise** échoue (`shouldMatchPetNameCaseInsensitively[2]`
> par exemple) — plutôt que de deviner laquelle des quatre méthodes copiées-collées est en cause.

### Comment lire `@CsvSource`

- Chaque **chaîne** du tableau représente **une exécution** de la méthode de test — ici, la méthode
  s'exécute 2 fois (2 lignes fournies), chacune avec ses propres valeurs.
- Chaque ligne est découpée par des **virgules** en autant de valeurs qu'il y a de **paramètres**
  dans la méthode, **dans le même ordre** : `"Léo, léo, true"` correspond à
  `nomEnregistré = "Léo"`, `recherche = "léo"`, `trouvé = true`.
- JUnit **convertit automatiquement** chaque valeur texte vers le type du paramètre correspondant :
  ici `"true"`/`"false"` deviennent des `boolean`, alors que `"Léo"`/`"léo"` restent des `String`.
  Cette conversion fonctionne pour les types simples (`int`, `boolean`, `double`, énumérations,
  etc.) — pas besoin de code supplémentaire.
- Les espaces après les virgules sont ignorés par défaut (`"Léo, léo, true"` équivaut à
  `"Léo,léo,true"`).
- Chaque ligne teste un **scénario distinct** : la première vérifie que la casse est bien ignorée
  (`"Léo"` trouvé en cherchant `"léo"` → `true`), la deuxième vérifie qu'un nom différent n'est
  **pas** trouvé (`"Milo"` cherché alors que le pet s'appelle `"Léo"` → `false`). C'est la même
  logique qu'un tableau de cas de test « entrée → sortie attendue », mais exécutée par JUnit sans
  boucle ni duplication manuelle.
- Dans le rapport, chaque ligne apparaît comme une exécution séparée et numérotée (par exemple
  `shouldMatchPetNameCaseInsensitively[1]`, `[2]`), ce qui permet de savoir immédiatement laquelle
  échoue si l'une des deux combinaisons casse.

---

## 5️⃣ Isoler une dépendance avec un mock

### Pourquoi un mock, et quand l'utiliser

Certaines classes ne peuvent pas être testées « en pur Java » comme `Owner` plus haut : elles
dépendent d'un objet difficile, coûteux ou risqué à construire réellement dans un test — une vraie
base de données, un vrai appel réseau, un vrai contexte de validation Jakarta. Un **mock** (via
Mockito, déjà inclus dans `spring-boot-starter-test`) remplace cette dépendance par une version
simulée, entièrement sous votre contrôle : vous décidez exactement ce qu'elle retourne, sans jamais
exécuter son vrai code.

| Situation | Objet réel ou mock ? |
|---|---|
| `Owner`, `Pet` — objets du domaine, logique pure, rapide, sans effet de bord | **Objet réel** (voir la section 1 plus haut) — inutile de mocker ce qui est déjà simple et rapide à construire |
| `ClinicService` dans un test de contrôleur — accède normalement à une vraie base de données | **Mock** — on veut tester la logique du contrôleur (validation, codes de statut HTTP), pas relancer une vraie base de données à chaque test |
| `ConstraintValidatorContext` — objet Jakarta Validation, non trivial à instancier hors d'un vrai contexte de validation | **Mock** — construire le vrai objet exigerait de démarrer toute l'infrastructure de validation pour tester une seule méthode |

> ⚠️ Un mock n'est **pas** une solution universelle. Mocker un objet simple comme `Owner` ajouterait
> de la complexité sans aucun bénéfice — pire, un mock mal configuré peut faire passer un test qui
> ne vérifie plus rien de réel. Réservez les mocks aux dépendances réellement difficiles à faire
> intervenir telles quelles dans un test.

### Cas d'usage 1 — simuler une valeur de retour

Le cas le plus simple : programmer ce qu'une méthode du mock doit retourner quand elle est appelée
avec certains arguments. Le projet utilise deux syntaxes strictement équivalentes pour ça, selon les
fichiers :

```java
// Style classique Mockito -- utilisé dans PetAgeValidatorTest
ConstraintValidatorContext context = mock(ConstraintValidatorContext.class);
when(context.buildConstraintViolationWithTemplate(anyString()))
        .thenReturn(mock(ConstraintValidatorContext.ConstraintViolationBuilder.class));
```

```java
// Style BDD (Behavior-Driven Development) -- utilisé dans OwnerRestControllerV1Tests
// (import static org.mockito.BDDMockito.given;)
given(this.clinicService.findOwnerById(1))
        .willReturn(ownerMapper.toOwner(owners.get(0)));
```

`when(x).thenReturn(y)` et `given(x).willReturn(y)` font **exactement** la même chose — seul le
vocabulaire change (`given`/`willReturn` est pensé pour se lire comme une phrase « Étant donné que
X, alors retourne Y », plus proche du style Given-When-Then). Utilisez celui déjà en place dans le
fichier que vous modifiez, par cohérence avec le reste de l'équipe.

### Cas d'usage 2 — simuler une exception (méthode `void`)

`testCreatePetShouldNotExposeTechnicalDetails`, dans le projet, illustre un cas plus subtil : on
veut vérifier que le contrôleur transforme une erreur technique de bas niveau (une contrainte de
base de données violée) en une réponse HTTP propre, **sans jamais exposer** le détail technique au
client de l'API :

```java
@Test
@WithMockUser(roles = "OWNER_ADMIN")
void testCreatePetShouldNotExposeTechnicalDetails() throws Exception {
    String technicalMessage =
        "could not execute statement; SQL [insert into pets ...]; constraint [fk_pet_owner]";
    given(this.clinicService.findOwnerById(1)).willReturn(ownerMapper.toOwner(owners.get(0)));

    // savePet(...) retourne void -- impossible d'écrire when(clinicService.savePet(pet))
    // puisqu'on ne peut pas passer un appel à une méthode void en argument de when().
    // Mockito fournit doThrow(...).when(mock).methode(...) pour ce cas précis :
    doThrow(new DataIntegrityViolationException(technicalMessage))
        .when(this.clinicService).savePet(any());

    this.mockMvc.perform(post("/api/owners/1/pets")
            .content(newPetAsJSON).contentType(MediaType.APPLICATION_JSON_VALUE))
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.detail")
            .value("The requested resource could not be processed due to a data constraint violation"));
}
```

> 💡 **Pourquoi `doThrow().when()` et pas `when().thenThrow()` ici ?** `savePet(Pet pet)` retourne
> `void` — on ne peut pas écrire `when(this.clinicService.savePet(pet))` puisque `savePet(pet)`
> lui-même ne retourne rien à passer à `when(...)`. Mockito inverse donc l'ordre pour les méthodes
> `void` : on déclare d'abord l'action (`doThrow(...)`), puis sur quel mock et quelle méthode elle
> s'applique (`.when(mock).methode(...)`). Utilisez `doThrow`/`doReturn`/`doNothing` chaque fois que
> la méthode simulée est `void` ou que `when(...)` seul poserait un problème de compilation.

Ce test illustre aussi une bonne pratique de sécurité au passage : `technicalMessage` (qui contient
le nom d'une table et d'une contrainte SQL) n'apparaît **jamais** dans la réponse HTTP vérifiée —
seul un message générique (`jsonPath("$.detail")`) est exposé au client. Le mock permet ici de
provoquer facilement, à volonté, un scénario d'erreur qu'il serait risqué ou compliqué de
reproduire avec une vraie base de données.

### Cas d'usage 3 — `any()` et les autres *argument matchers*

`any()` (et ses variantes `anyString()`, `anyInt()`, `eq(valeur)`, etc.) indique à Mockito
« n'importe quelle valeur, peu importe laquelle, appliquer ce comportement ». C'est utile quand
seul l'appel de la méthode compte pour ce test, pas la valeur exacte de son argument :

```java
doThrow(new DataIntegrityViolationException(technicalMessage))
    .when(this.clinicService).savePet(any());   // n'importe quel Pet passé à savePet(...)
```

> ⚠️ **Piège fréquent** : si un mock a plusieurs arguments, **soit tous sont des valeurs précises,
> soit tous doivent être des matchers** (`any()`, `eq()`, etc.) — impossible de mélanger les deux
> dans un même appel (ex. `savePet(any(), 1)` ne compile pas s'il faut aussi préciser un second
> argument précis; il faudrait écrire `savePet(any(), eq(1))`). Mockito lance une erreur explicite
> à l'exécution si cette règle est violée — un message qu'il faut savoir reconnaître plutôt que de
> paniquer devant.

### Cas d'usage 4 — `verify()` : confirmer qu'une méthode a bien été appelée

Un mock ne sert pas seulement à fournir une valeur de retour — il permet aussi de vérifier
**après coup** qu'une méthode a bien été appelée, et avec quels arguments. C'est `verify()`.

Fait révélateur : `testDeleteOwnerSuccess`, dans le projet, vérifie seulement le code de statut HTTP
retourné (`204 No Content`) :

```java
@Test
@WithMockUser(roles = "OWNER_ADMIN")
void testDeleteOwnerSuccess() throws Exception {
    final Owner owner = ownerMapper.toOwner(owners.get(0));
    given(this.clinicService.findOwnerById(1)).willReturn(owner);

    this.mockMvc.perform(delete("/api/owners/1")
            .content(newOwnerAsJSON).contentType(MediaType.APPLICATION_JSON_VALUE))
        .andExpect(status().isNoContent());
    // ⚠️ Ce test s'arrête ici -- rien ne confirme que clinicService.deleteOwner(owner)
    // a réellement été appelé, alors que c'est exactement ce que OwnerRestControllerV1
    // .deleteOwner(...) est censé faire.
}
```

Ce test passerait **encore au vert** même si quelqu'un supprimait accidentellement l'appel à
`this.clinicService.deleteOwner(owner);` dans le contrôleur (tant que le contrôleur renvoie quand
même `204 No Content`) — un vrai trou dans le filet de sécurité. `verify()` comble ce trou :

```java
@Test
@WithMockUser(roles = "OWNER_ADMIN")
void testDeleteOwnerSuccess() throws Exception {
    final Owner owner = ownerMapper.toOwner(owners.get(0));
    given(this.clinicService.findOwnerById(1)).willReturn(owner);

    this.mockMvc.perform(delete("/api/owners/1")
            .content(newOwnerAsJSON).contentType(MediaType.APPLICATION_JSON_VALUE))
        .andExpect(status().isNoContent());

    // Confirme que la suppression a RÉELLEMENT été demandée au service, pas seulement
    // que le contrôleur a répondu 204 par coïncidence.
    verify(this.clinicService).deleteOwner(owner);
}
```

| Variante de `verify` | Vérifie que... |
|---|---|
| `verify(mock).methode(args)` | La méthode a été appelée **exactement une fois**, avec ces arguments |
| `verify(mock, times(2)).methode(args)` | La méthode a été appelée **exactement 2 fois** |
| `verify(mock, never()).methode(args)` | La méthode n'a **jamais** été appelée |
| `verify(mock, atLeastOnce()).methode(args)` | La méthode a été appelée **au moins une fois** |

<details>
<summary>🤔 Testez-vous</summary>

Un test utilise `given(this.clinicService.findOwnerById(1)).willReturn(owner)` mais n'appelle
jamais `verify(...)` sur `clinicService`. Un développeur supprime par erreur la logique qui
sauvegarde réellement les changements dans le contrôleur, mais le test continue de passer. Pourquoi,
et comment corriger le test ?

**Réponse** : configurer un mock avec `given()`/`when()` ne vérifie **rien** en soi — ça prépare
seulement une valeur de retour au cas où la méthode serait appelée. Si le test ne vérifie que le
code de statut HTTP final, il peut rester vert même si un appel important au service a été retiré
par erreur, tant que la réponse HTTP reste plausible. Ajouter `verify(this.clinicService).save...(...)`
force le test à confirmer que l'appel a réellement eu lieu, pas seulement que la réponse « a l'air »
correcte.
</details>

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Trouvez, dans les tests de votre projet, un test qui configure un mock (`given`/`when`) mais qui
n'appelle jamais `verify(...)` sur ce même mock. Ajoutez le `verify(...)` manquant, puis cassez
volontairement le code de production (retirez l'appel réel au service) pour confirmer que votre test
enrichi échoue désormais — alors qu'il ne l'aurait pas fait avant votre ajout.
{{% /notice %}}

---


## 6️⃣ Cycle de vie d'une classe de test

Répéter la construction d'un objet identique dans chaque méthode de test (comme `new Owner()`
ci-dessus) devient répétitif. Les annotations de cycle de vie permettent de factoriser cette
préparation :

```java
class OwnerTest {

    private Owner owner;

    @BeforeEach
    void setUp() {
        // Exécuté avant CHAQUE test : état propre garanti, aucun risque de fuite entre tests
        owner = new Owner();
        Pet pet = new Pet();
        pet.setName("Léo");
        owner.addPet(pet);
    }

    @Test
    void shouldFindPetByNameIgnoringCase() {
        Pet found = owner.getPet("léo");
        assertNotNull(found);
    }

    @Test
    void shouldReturnNullWhenPetNameDoesNotExist() {
        Pet found = owner.getPet("Inconnu");
        assertNull(found);
    }

    @AfterEach
    void tearDown() {
        // Exécuté après CHAQUE test : nettoyage si nécessaire (fichiers temporaires, etc.)
    }
}
```

| Annotation | Quand elle s'exécute |
|---|---|
| `@BeforeEach` | Avant **chaque** méthode `@Test` de la classe |
| `@AfterEach` | Après **chaque** méthode `@Test` de la classe |
| `@BeforeAll` | **Une seule fois**, avant tous les tests de la classe (méthode `static`) |
| `@AfterAll` | **Une seule fois**, après tous les tests de la classe (méthode `static`) |

> ⚠️ `@BeforeEach` recrée `owner` avant **chaque** test — c'est ce qui garantit l'**indépendance**
> des tests (aucun état résiduel d'un test à l'autre), un principe qui sera approfondi plus tard
> dans la session. Sans cette réinitialisation, un test qui modifie `owner` pourrait fausser le
> résultat du test suivant, selon l'ordre d'exécution.

---

## 7️⃣ Où placer vos tests

Convention Maven stricte : le code de production vit dans `src/main/java/...`, les tests dans
`src/test/java/...`, en respectant le **même** package. Une classe de test pour
`org.springframework.samples.petclinic.model.Owner` doit donc se trouver dans
`src/test/java/org/springframework/samples/petclinic/model/OwnerTest.java` — jamais dans
`src/main/`, où elle serait empaquetée par erreur dans le livrable final.

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Repérez une classe de votre projet qui n'a **aucun** test, et pour laquelle vous pouvez écrire un
test **sans** avoir besoin de mock ni de contexte Spring (comme `Owner` ci-dessus). Écrivez une
première classe de test avec au moins deux méthodes `@Test`.
{{% /notice %}}

---

## 📚 Références

- Documentation officielle JUnit — [Writing Tests](https://docs.junit.org/current/user-guide/#writing-tests),
  guide de référence pour les annotations et assertions présentées ici (JUnit 6, moteur Jupiter).
- Documentation officielle JUnit — [Parameterized Tests](https://docs.junit.org/current/user-guide/#writing-tests-parameterized-tests).
- Documentation officielle Mockito — [site.mockito.org](https://site.mockito.org/) — bases de
  `mock()`/`when()`/`thenReturn()` illustrées ci-dessus.
- Kent Beck (2002), *Test-Driven Development: By Example*, Addison-Wesley.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice JUnit]({{% relref "/semaine-4/exercice-junit" %}}) pour écrire vos
premiers tests directement sur votre projet.
{{% /notice %}}
