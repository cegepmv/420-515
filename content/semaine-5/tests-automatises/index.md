+++
title = "Tests automatisés — unitaires, régression, couverture, bonnes pratiques"
weight = 1
draft = true
+++

En semaine 4, vous avez écrit vos premiers tests JUnit. Cette semaine approfondit **pourquoi** on
écrit certains tests plutôt que d'autres, **comment** mesurer objectivement l'effort de test déjà
fourni, et **quelles règles** suivre pour qu'une suite de tests reste un atout plutôt qu'un fardeau.

---

## 1️⃣ Test unitaire vs test de régression : même outil (JUnit), deux intentions différentes

Un **test unitaire** vérifie qu'une unité de code (une méthode, une classe) se comporte comme
prévu, en isolant ses dépendances. Il est généralement écrit **en même temps** que le code, pour
documenter et valider son comportement attendu.

Un **test de régression** a une intention différente : il capture, sous forme de test automatisé,
un bogue **déjà observé**, pour garantir qu'il ne réapparaîtra jamais silencieusement. La séquence
typique est :

1. Un bogue est signalé ou découvert.
2. On écrit un test qui reproduit ce bogue — **ce test doit échouer** tant que le bogue n'est pas
   corrigé (sinon il ne prouve rien).
3. On corrige le bogue.
4. Le même test passe maintenant au vert — preuve tangible et automatisée que le bogue est réglé.
5. Ce test reste dans la suite **pour toujours** : si quelqu'un réintroduit accidentellement ce
   bogue (ex. en réusinant le code), le test échoue immédiatement, en local ou en CI.

> 💡 Un test de régression est donc, techniquement, un test unitaire (même syntaxe JUnit, mêmes
> assertions) — ce qui change, c'est **l'origine** du test (un bogue réel plutôt qu'une spécification
> anticipée) et son rôle dans le temps (empêcher un retour en arrière, plutôt que documenter un
> comportement neuf).

### Exemple réel — un test de régression qui a cessé de protéger le projet

`PetAgeValidator` (validation de la date de naissance d'un animal) illustre bien le risque qu'un
test de régression est censé éliminer :

```java
// PetAgeValidator.java
@Override
public boolean isValid(LocalDate birthDate, ConstraintValidatorContext context) {
    if (birthDate == null) {
        return true;
    }
    LocalDate today = LocalDate.now();
    if (birthDate.isAfter(today)) {
        context.disableDefaultConstraintViolation();
        context.buildConstraintViolationWithTemplate(
            "Birth date cannot be in the future"
        ).addConstraintViolation();
        return false;
    }
    if (birthDate.isBefore(today.minusYears(100))) {
        context.disableDefaultConstraintViolation();
        context.buildConstraintViolationWithTemplate(
            "Birth date cannot be older than 50 years"   // ⚠️ message désynchronisé de la règle réelle
        ).addConstraintViolation();
        return false;
    }
    return true;
}
```

Et le test qui l'accompagne, dans `PetAgeValidatorTest` :

```java
// PetAgeValidatorTest.java
@Disabled("Bogue intentionnel A26 - voir README-COURS-A26.md - à réactiver au moment choisi en classe")
@Test
void shouldReturnFalseWhenBirthDateIsOlderThan50Years() {
    LocalDate tooOldDate = LocalDate.now().minusYears(51);

    ConstraintValidatorContext context = mock(ConstraintValidatorContext.class);
    when(context.buildConstraintViolationWithTemplate(anyString()))
            .thenReturn(mock(ConstraintValidatorContext.ConstraintViolationBuilder.class));

    boolean result = validator.isValid(tooOldDate, context);

    assertFalse(result);
}
```

Ce test a été écrit à une époque où la règle métier limitait l'âge à **50 ans** (`isBefore(today.minusYears(50))`).
Le code a ensuite évolué : la limite réelle est passée à **100 ans**, mais ni le message d'erreur
(toujours « ... older than 50 years ») ni ce test n'ont été mis à jour en conséquence. Résultat :
une date vieille de 51 ans est aujourd'hui **valide** selon le code, mais ce test s'attend encore à
ce qu'elle soit rejetée — il échouerait s'il n'était pas désactivé (`@Disabled`).

> ⚠️ C'est exactement le scénario qu'un test de régression est censé prévenir : une règle métier a
> changé sans que la suite de tests suive, et personne ne s'en est rendu compte parce que le test
> concerné a simplement été neutralisé plutôt que corrigé ou retiré consciemment. Un `@Disabled`
> qui traîne est un signal d'alarme à ne jamais ignorer en revue de code — c'est un trou dans le
> filet de sécurité, invisible dans les résultats verts habituels.

<details>
<summary>🤔 Testez-vous</summary>

Vous retirez l'annotation `@Disabled` de `shouldReturnFalseWhenBirthDateIsOlderThan50Years` et
relancez les tests. Le test échoue. Qu'est-ce que cela révèle : un bogue dans `PetAgeValidator`, un
bogue dans le test, ou les deux ?

**Réponse** : le test est désynchronisé avec la règle métier actuelle (100 ans, pas 50) — c'est le
**test** qui est obsolète, pas nécessairement le comportement du validateur. Mais le **message
d'erreur** du validateur, lui, ment toujours sur la vraie limite (100 ans) : c'est un vrai bogue
séparé, à corriger dans le code de production. Corriger un test qui échoue exige toujours de se
demander lequel des deux — code ou test — reflète la véritable intention, avant de modifier l'un
ou l'autre à l'aveugle.
</details>

---

## 2️⃣ Couverture de code : ce qu'elle mesure, et ce qu'elle ne mesure pas

La **couverture de code** (*code coverage*) quantifie la proportion du code source réellement
exécutée pendant l'exécution d'une suite de tests. Les métriques les plus courantes :

| Métrique | Ce qu'elle compte |
|---|---|
| Couverture de lignes (*line coverage*) | % de lignes exécutées au moins une fois |
| Couverture de branches (*branch coverage*) | % de branches (`if`/`else`, `switch`) empruntées, dans les deux sens |
| Couverture de méthodes | % de méthodes appelées au moins une fois |

### Ajouter JaCoCo au projet

Le projet du cours n'a **pas encore** d'outil de mesure de couverture configuré — c'est justement
l'objet de l'exercice de cette semaine. **JaCoCo** (*Java Code Coverage*) est l'outil standard de
l'écosystème Java/Maven. Extrait de configuration à ajouter dans la section `<build><plugins>` du
`pom.xml` :

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

- `prepare-agent` s'attache au processus JVM **avant** l'exécution des tests, pour observer quelles
  lignes s'exécutent.
- `report` génère, après `mvn test`, un rapport HTML consultable dans
  `target/site/jacoco/index.html`.

```bash
mvn clean test
# puis ouvrir target/site/jacoco/index.html dans un navigateur
```

### ⚠️ Ce qu'une couverture élevée NE garantit PAS

> 💡 **100 % de couverture de lignes ne veut pas dire "0 bogue".** Une ligne peut être *exécutée*
> par un test sans que son résultat soit *vérifié* par une assertion pertinente.

Exemple concret sur `PetAgeValidator` : un test comme celui-ci exécute la ligne fautive (le message
« 50 years ») sans jamais la couvrir réellement du point de vue métier :

```java
@Test
void callsIsValidWithoutCheckingTheRule() {
    LocalDate someDate = LocalDate.now().minusYears(60);
    validator.isValid(someDate, mock(ConstraintValidatorContext.class));
    // Aucune assertion sur le résultat : la ligne s'exécute, la "couverture" grimpe,
    // mais rien n'est réellement vérifié.
}
```

Ce test ferait grimper la couverture de lignes sans jamais détecter que le message d'erreur ment
sur la vraie limite d'âge. La couverture est donc un indicateur **d'effort minimal de test** (« estce
qu'on a complètement oublié de tester ce code ? »), pas une preuve de qualité ou d'absence de
bogue.

<details>
<summary>🤔 Testez-vous</summary>

Une classe affiche 95 % de couverture de lignes dans le rapport JaCoCo. Peut-on affirmer que cette
classe ne contient presque aucun bogue ?

**Réponse** : non. La couverture mesure seulement quelles lignes ont été **exécutées** pendant les
tests, pas si leurs résultats ont été **vérifiés correctement** par des assertions pertinentes. Un
code peut être entièrement exécuté par des tests mal conçus (sans assertion, ou avec une assertion
qui ne teste pas le bon comportement) et rester truffé de bogues malgré une couverture élevée.
</details>

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Ajoutez le plugin JaCoCo à votre `pom.xml`, lancez `mvn clean test`, et ouvrez le rapport HTML.
Identifiez une classe avec une couverture de branches nettement plus faible que sa couverture de
lignes — que cela révèle-t-il typiquement (indice : des `if`/`else` où un seul des deux chemins est
testé) ?
{{% /notice %}}

---

## 3️⃣ Bonnes pratiques d'écriture de tests

### Nommage clair

Un nom de test devrait décrire **le scénario** et **le résultat attendu**, sans avoir à lire le
corps du test. Le projet suit déjà cette convention (`should<Résultat>When<Scénario>`) :

```java
void shouldReturnFalseWhenBirthDateIsInFuture()
void shouldReturnTrueWhenBirthDateIsValid()
void shouldReturnTrueWhenBirthDateIsNull()
```

> ⚠️ À éviter : `test1()`, `testValidator()`, `testIsValid()` — ces noms n'indiquent ni le scénario
> ni le résultat attendu; il faut lire tout le corps du test pour comprendre ce qui est vérifié.

### Structure Arrange-Act-Assert (AAA)

Chaque test gagne en lisibilité s'il suit systématiquement trois étapes distinctes, comme dans
`shouldReturnFalseWhenBirthDateIsInFuture` :

```java
@Test
void shouldReturnFalseWhenBirthDateIsInFuture() {
    // Arrange : préparer les données et les objets nécessaires
    LocalDate futureDate = LocalDate.now().plusDays(1);
    ConstraintValidatorContext context = mock(ConstraintValidatorContext.class);
    when(context.buildConstraintViolationWithTemplate(anyString()))
            .thenReturn(mock(ConstraintValidatorContext.ConstraintViolationBuilder.class));

    // Act : exécuter le comportement à tester
    boolean result = validator.isValid(futureDate, context);

    // Assert : vérifier le résultat
    assertFalse(result);
}
```

### Un seul comportement testé par test

Un test qui vérifie plusieurs comportements sans lien devient difficile à diagnostiquer quand il
échoue (lequel des trois comportements a cassé ?). Le fichier `PetAgeValidatorTest` illustre la
bonne pratique : un test séparé pour chaque scénario (date future, date trop ancienne, date valide,
date nulle), plutôt qu'un seul test géant qui enchaîne les quatre cas avec quatre assertions.

### Indépendance des tests

Un test ne devrait **jamais** dépendre de l'exécution préalable d'un autre test, ni d'un état
partagé mutable entre les tests. Deux signes fréquents de violation :

- Un test qui échoue **seulement** quand on exécute toute la classe, mais qui passe si on
  l'exécute seul.
- Un champ d'instance modifié par un test et lu par un autre, sans réinitialisation entre les deux
  (`@BeforeEach` sert justement à repartir d'un état propre à chaque test).

<details>
<summary>🤔 Testez-vous</summary>

Un test passe quand vous l'exécutez seul (clic droit → Run), mais échoue quand vous exécutez toute
la classe de tests. Quelle est la cause la plus probable ?

**Réponse** : une dépendance cachée entre tests — un état partagé (champ d'instance, base de
données en mémoire, fichier temporaire) modifié par un autre test qui s'exécute avant lui, sans
réinitialisation propre entre chaque test. Les tests doivent être écrits pour être exécutables dans
n'importe quel ordre, isolément ou ensemble.
</details>

---

## 📚 Références

- Kent Beck (2002), *Test-Driven Development: By Example*, Addison-Wesley — origine du cycle
  Arrange-Act-Assert (souvent présenté sous le nom Given-When-Then).
- Documentation officielle JaCoCo — [jacoco.org](https://www.jacoco.org/jacoco/trunk/doc/) — guide
  de configuration Maven du plugin de couverture.
- Martin Fowler, *Regression Test* — [martinfowler.com/bliki/RegressionTest.html](https://martinfowler.com/bliki/RegressionTest.html).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Tests automatisés]({{% relref "/semaine-5/exercice-tests-automatises" %}})
pour appliquer ces notions directement sur votre projet.
{{% /notice %}}
