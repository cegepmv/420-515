+++
title = "Maven (2/2) — cycle de build et exécution des tests"
weight = 1
+++

La semaine dernière (Maven 1/2), vous avez exploré le `pom.xml` : dépendances, *scope*, plugins, et
le cycle de vie complet (`validate` → ... → `deploy`). Cette semaine se concentre sur **une seule
phase** de ce cycle — `test` — et sur les outils qui l'exécutent concrètement.

---

## 1️⃣ `maven-surefire-plugin` : qui exécute vos tests

Quand vous lancez `mvn test`, ce n'est pas Maven lui-même qui exécute vos classes de test — c'est le
plugin **Surefire**, invoqué automatiquement pendant la phase `test`. Il recherche par convention
toutes les classes dont le nom correspond à un patron précis :

- `**/*Test.java`
- `**/Test*.java`
- `**/*Tests.java`
- `**/*TestCase.java`

C'est exactement pour cette raison que les classes de test du projet portent des noms comme
`PetAgeValidatorTest`, `ValidatorTests` ou `OwnerRestControllerV1Tests` — une classe nommée
`OwnerHelper.java` (sans suffixe reconnu) serait **silencieusement ignorée** par Surefire, même si
elle contient des méthodes annotées `@Test`.

```bash
mvn test
```

Extrait réel de la sortie console :

```
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running org.springframework.validation.PetAgeValidatorTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0.045 s
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 47, Failures: 0, Errors: 0, Skipped: 1
```

> 💡 Le `Skipped: 1` correspond ici à un test annoté `@Disabled` dans `PetAgeValidatorTest` —
> Surefire le compte, mais ne l'exécute pas.

### Rapports Surefire

Après `mvn test`, chaque classe de test produit un rapport dans `target/surefire-reports/` :

- `TEST-org.springframework.validation.PetAgeValidatorTest.xml` (format XML, lisible par des outils
  comme le pipeline CI ou un plugin d'IDE).
- `org.springframework.validation.PetAgeValidatorTest.txt` (résumé lisible directement) :

```
-------------------------------------------------------------------------------
Test set: org.springframework.validation.PetAgeValidatorTest
-------------------------------------------------------------------------------
Tests run: 3, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0.045 s
```

<details>
<summary>🤔 Testez-vous</summary>

Vous créez une classe `HelperOwnerChecks.java` contenant plusieurs méthodes annotées `@Test`. Après
`mvn test`, aucune trace de cette classe n'apparaît dans la console ni dans
`target/surefire-reports/`. Pourquoi ?

**Réponse** : le nom de la classe ne correspond à aucun des patrons reconnus par
`maven-surefire-plugin` (`*Test`, `Test*`, `*Tests`, `*TestCase`) — Surefire ne la considère donc
jamais comme une classe de test, peu importe son contenu. Il faut la renommer, par exemple
`OwnerChecksTest.java`.
</details>

---

## 2️⃣ Cibler un test précis avec `-Dtest`

Exécuter toute la suite à chaque petite modification devient lent sur un gros projet. Le paramètre
`-Dtest` permet de cibler précisément :

```bash
# une seule classe
mvn test -Dtest=PetAgeValidatorTest

# une seule méthode d'une classe
mvn test -Dtest=PetAgeValidatorTest#shouldReturnTrueWhenBirthDateIsValid

# plusieurs classes, séparées par une virgule
mvn test -Dtest=PetAgeValidatorTest,ValidatorTests

# un patron avec caractère générique
mvn test -Dtest=*ValidatorTest*
```

---

## 3️⃣ Ignorer les tests : deux options, deux effets très différents

| Commande | Effet |
|---|---|
| `mvn install -DskipTests` | **Compile** les classes de test, mais ne les **exécute pas** |
| `mvn install -Dmaven.test.skip=true` | Ne compile **ni** n'exécute les classes de test |

> ⚠️ `-Dmaven.test.skip=true` est plus radical : si votre code de test ne compile même plus (ex.
> après un renommage de méthode dans le code de production), cette option masque complètement le
> problème — le build réussit alors que la suite de tests est en réalité cassée. `-DskipTests` est
> presque toujours le choix le plus sûr : au minimum, il révèle une erreur de compilation dans les
> tests, même s'il n'exécute pas leurs assertions.

<details>
<summary>🤔 Testez-vous</summary>

Un coéquipier renomme une méthode utilisée par plusieurs tests, sans mettre les tests à jour, puis
lance `mvn install -Dmaven.test.skip=true` pour livrer rapidement. Le build réussit. Que risque de
découvrir la prochaine personne qui lance `mvn test` normalement ?

**Réponse** : les tests ne compileront probablement plus (erreur de compilation, pas seulement un
test qui échoue), puisque `-Dmaven.test.skip=true` a empêché Maven de même essayer de les compiler
pendant la livraison précédente. Le problème n'a pas été détecté plus tôt uniquement parce que
cette option cache complètement le code de test du build.
</details>

---

## 4️⃣ Tests unitaires vs tests d'intégration : Surefire vs Failsafe

Maven distingue, par convention, deux plugins pour deux types de tests, rattachés à deux phases
différentes du cycle de vie :

| Plugin | Phase | Convention de nommage | Type de test |
|---|---|---|---|
| `maven-surefire-plugin` | `test` | `*Test`, `*Tests` | Tests unitaires — rapides, sans dépendances externes réelles (base de données, réseau) |
| `maven-failsafe-plugin` | `verify` (après `package`) | `*IT` (*Integration Test*) | Tests d'intégration — plus lents, avec de vraies dépendances (ex. une vraie base de données) |

> 💡 Pourquoi deux plugins distincts ? Failsafe garantit que même si un test **d'intégration**
> échoue, le rapport final n'est produit qu'**après** que tous les tests d'intégration se soient
> exécutés (contrairement à Surefire, qui arrête le build dès le premier échec en phase `test`) —
> utile pour ne pas gaspiller un environnement de test coûteux (ex. une base de données démarrée
> spécialement pour l'occasion) sur un seul échec précoce.

Le projet du cours illustre bien cette distinction : `ClinicServiceJpaTests`,
`ClinicServiceH2JdbcTests` et `ClinicServiceHsqlJdbcTests` portent le suffixe `*Tests` (convention
Surefire) même s'ils touchent une vraie base de données en mémoire — un rappel que la frontière
« unitaire / intégration » dépend surtout de ce que le test **exerce réellement**, pas seulement de
son nom.

---

## 5️⃣ Filtrer par tags (aperçu)

JUnit 6 (Jupiter) permet d'étiqueter des tests avec `@Tag` pour les exécuter sélectivement — utile plus tard
dans la session pour, par exemple, exclure des tests lents d'un pipeline CI rapide :

```java
@Tag("lent")
@Test
void shouldHandleLargeDataset() { /* ... */ }
```

```bash
mvn test -Dgroups="lent"           # exécute SEULEMENT les tests tagués "lent"
mvn test -DexcludedGroups="lent"   # exécute tout SAUF les tests tagués "lent"
```

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Lancez `mvn test -Dtest=PetAgeValidatorTest` sur votre projet, puis consultez le rapport généré dans
`target/surefire-reports/`. Combien de tests ont été exécutés, et combien ont été ignorés (Skipped) ?
{{% /notice %}}

---

## 📚 Références

- Documentation officielle Apache Maven Surefire —
  [maven.apache.org/surefire/maven-surefire-plugin](https://maven.apache.org/surefire/maven-surefire-plugin/).
- Documentation officielle Apache Maven Failsafe —
  [maven.apache.org/surefire/maven-failsafe-plugin](https://maven.apache.org/surefire/maven-failsafe-plugin/).
- Documentation officielle JUnit — [docs.junit.org](https://docs.junit.org/current/user-guide/) (JUnit 6, moteur Jupiter).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Maven — exécution des tests]({{% relref "/semaine-4/exercice-maven-tests" %}}).
{{% /notice %}}
