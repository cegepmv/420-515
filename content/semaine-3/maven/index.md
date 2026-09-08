+++
title = "Maven (1/2) — pom.xml, dépendances, plugins"
weight = 3
+++

## Pourquoi un outil de build ?

Compiler, gérer les librairies externes et empaqueter un projet Java « à la main » devient vite
ingérable : chaque librairie a ses propres dépendances (dépendances *transitives*), parfois en
conflit de version entre elles. **Maven** standardise tout ça avec un fichier de configuration
unique — `pom.xml` — et une convention commune, reconnue dans la quasi-totalité des projets Java
d'entreprise.

---

## 1️⃣ Anatomie du `pom.xml`

**Exemple — extrait réel du `pom.xml` de `spring-petclinic-rest`** (le projet utilisé dans ce
cours) :

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.springframework.samples</groupId>
    <artifactId>spring-petclinic-rest</artifactId>
    <version>4.0.2</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.1.0</version>
        <relativePath/> <!-- lookup parent from Maven repository -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!-- ... autres dépendances ... -->
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <!-- ... -->
            </plugin>
        </plugins>
    </build>
</project>
```

| Élément | Rôle |
|---|---|
| `groupId` | Identifiant de l'organisation/projet (convention : nom de domaine inversé) — ici `org.springframework.samples` |
| `artifactId` | Nom du module/projet lui-même — ici `spring-petclinic-rest` |
| `version` | Version du projet — ici `4.0.2` (figée, contrairement à un `-SNAPSHOT` « en développement ») |
| `parent` | Un `pom.xml` **parent** dont on hérite (versions de dépendances, plugins, configuration commune) — ici `spring-boot-starter-parent`, le parent standard de tout projet Spring Boot |
| `dependencies` | Librairies externes requises |
| `build > plugins` | Outils qui interviennent pendant le build (compiler, tester, empaqueter...) |

> 💡 Grâce au `parent` (`spring-boot-starter-parent`), la plupart des dépendances Spring n'ont
> **pas besoin** de préciser leur `<version>` : c'est le parent qui l'impose, pour garantir que
> toutes les librairies Spring d'un même projet sont compatibles entre elles. C'est pourquoi vous
> ne voyez aucune balise `<version>` sur `spring-boot-starter-actuator` ci-dessus.

### Autres `pom.xml` réels — le cas du **multi-module**

Le `pom.xml` de `spring-petclinic-rest` ci-dessus est un `pom.xml` **de module unique**
(`packaging` implicite : `jar`). Beaucoup de projets Java réels sont plutôt organisés en
**plusieurs modules** qui partagent un `pom.xml` **parent** commun. Deux exemples réels et
largement utilisés en production :

**[Gson](https://github.com/google/gson)** (bibliothèque JSON de Google, des milliards
d'utilisations) — extrait réel de son `pom.xml` racine :

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.google.code.gson</groupId>
    <artifactId>gson-parent</artifactId>
    <version>2.14.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>gson</module>
        <module>test-jpms</module>
        <module>extras</module>
        <module>metrics</module>
        <module>proto</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>4.13.2</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

**[Apache Maven](https://github.com/apache/maven)** lui-même (l'outil que vous utilisez est
construit... avec Maven) — même structure, avec en plus un `parent` :

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.apache.maven</groupId>
        <artifactId>maven-parent</artifactId>
        <version>49</version>
        <relativePath/>
    </parent>

    <artifactId>maven</artifactId>
    <version>4.1.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>api</module>
        <module>impl</module>
        <module>compat</module>
        <module>apache-maven</module>
    </modules>
</project>
```

Ce qui change par rapport à `spring-petclinic-rest` :

| Élément | Rôle dans un `pom.xml` parent multi-module |
|---|---|
| `<packaging>pom</packaging>` | Ce `pom.xml` **n'est pas un module compilable** — c'est un « chef d'orchestre » qui agrège d'autres modules. Sans cette balise, `packaging` vaudrait `jar` par défaut, ce qui n'a pas de sens pour un module qui ne contient aucun code. |
| `<modules>` | Liste des sous-dossiers (chacun avec son propre `pom.xml`) que Maven doit construire, dans l'ordre déterminé automatiquement par leurs dépendances entre eux. Une commande `mvn install` lancée à la racine construit **tous** les modules listés. |
| `<dependencyManagement>` | Centralise les **versions** de dépendances pour tous les modules enfants, sans forcer chaque module à les redéclarer — chaque module enfant peut alors écrire `<dependency>` sans `<version>`, exactement comme `spring-petclinic-rest` le fait pour ses dépendances Spring (héritées de `spring-boot-starter-parent`). |

> 💡 Un `pom.xml` avec `<parent>` **et** un projet Spring Boot comme le vôtre utilisent le même
> mécanisme d'héritage — la seule différence est qui joue le rôle de parent : `spring-boot-starter-parent`
> (fourni par Spring) contre `maven-parent`/`gson-parent` (défini par le projet lui-même dans un
> multi-module).

---

## 2️⃣ Dépendances et *scope*

Le `scope` d'une dépendance détermine **quand** elle est disponible :

| Scope | Disponible pour... | Exemple typique | Exemple réel dans `spring-petclinic-rest` |
|---|---|---|---|
| `compile` *(défaut)* | Compilation ET exécution ET tests | Une librairie utilisée partout (ex. Spring) | `spring-boot-starter-actuator` (pas de `<scope>` = `compile`) |
| `test` | Seulement la compilation/exécution des tests | JUnit, Mockito | `mockito-core`, `spring-boot-starter-security-test` |
| `provided` | Compilation, mais fournie par l'environnement d'exécution (pas empaquetée) | API Servlet sur un serveur qui la fournit déjà | — |
| `runtime` | Exécution seulement, pas nécessaire pour compiler | Un pilote JDBC | `h2`, `hsqldb`, `mysql-connector-j`, `postgresql` — **quatre** pilotes de base de données différents, tous en `runtime` |

> 💡 Pourquoi `spring-petclinic-rest` déclare-t-il **quatre** pilotes de base de données
> (`h2`, `hsqldb`, `mysql-connector-j`, `postgresql`) ? Parce que l'application peut être lancée
> avec différents profils Spring (`h2`, `hsqldb`, `mysql`, `postgres`) selon la base de données
> réellement utilisée en production ou en local. Chaque pilote est en scope `runtime` : le code
> compile sans eux (on ne référence jamais directement une classe `org.postgresql.*` dans le
> code métier — c'est le driver JDBC générique qui s'en charge), mais un seul sera nécessaire *à
> l'exécution*, selon le profil actif.

> ⚠️ Une erreur fréquente : mettre une dépendance de test (ex. JUnit) en `compile` par défaut —
> elle se retrouve alors inutilement empaquetée dans le livrable final.

### Dépendances transitives et conflits de version

Si votre projet dépend de la librairie A, et que A dépend elle-même de la librairie C en version
1.0, mais que votre projet dépend *aussi directement* de C en version 2.0 — lequel des deux Maven
utilise-t-il ? Maven applique la règle du **plus proche dans l'arbre** (*nearest wins*) : la
version déclarée le plus près de votre propre `pom.xml` l'emporte.

```bash
mvn dependency:tree
```

Extrait réel (raccourci) de la sortie sur `spring-petclinic-rest` :

```
[INFO] org.springframework.samples:spring-petclinic-rest:jar:4.0.2
[INFO] +- org.springframework.boot:spring-boot-starter-actuator:jar:4.1.0:compile
[INFO] +- com.h2database:h2:jar:2.4.240:runtime
[INFO] +- org.hsqldb:hsqldb:jar:2.7.3:runtime
[INFO] +- com.mysql:mysql-connector-j:jar:9.7.0:runtime
[INFO] +- org.postgresql:postgresql:jar:42.7.11:runtime
[INFO] \- org.springframework.data:spring-data-jdbc-core:jar:1.2.1.RELEASE:compile
[INFO]    \- (org.springframework:* omitted : exclu explicitement par le pom.xml)
```

Le dernier bloc illustre une **exclusion explicite** : `spring-data-jdbc-core` dépend
transitivement de plusieurs artefacts `org.springframework:*`, mais le `pom.xml` du projet les
exclut délibérément (via `<exclusions>`) pour éviter qu'ils entrent en conflit avec les versions
déjà imposées par `spring-boot-starter-parent` :

```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jdbc-core</artifactId>
    <version>${spring-data-jdbc.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework</groupId>
            <artifactId>*</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

`mvn dependency:tree` est l'outil de diagnostic à réflexe immédiat devant tout comportement
inattendu lié à une librairie — que ce soit un conflit de version « nearest wins » ou une
dépendance transitive indésirable qu'il faut exclure explicitement.

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Lancez `mvn dependency:tree` sur votre projet. Trouvez une dépendance transitive (une ligne
indentée qui n'est PAS déclarée directement dans votre `pom.xml`) et identifiez quelle dépendance
directe l'a introduite.
{{% /notice %}}

---

## 3️⃣ Plugins

Un **plugin** Maven exécute une tâche précise pendant une phase du build. Quelques plugins
omniprésents :

| Plugin | Rôle |
|---|---|
| `maven-compiler-plugin` | Compile le code source (`.java` → `.class`) |
| `maven-surefire-plugin` | Exécute les tests unitaires pendant la phase `test` |
| `maven-jar-plugin` / `spring-boot-maven-plugin` | Empaquette le livrable final (`.jar`) |
| `maven-failsafe-plugin` | Exécute les tests d'intégration (phase `verify`, distincte des tests unitaires) |

---

## 4️⃣ Cycle de vie Maven

![Cycle de vie Maven](images/cycle-vie-maven.svg)

| Phase | Ce qui se passe |
|---|---|
| `validate` | Vérifie que le projet est correct et toute l'information nécessaire est disponible |
| `compile` | Compile le code source |
| `test` | Exécute les tests unitaires (`maven-surefire-plugin`) |
| `package` | Empaquette le code compilé (`.jar`/`.war`) |
| `verify` | Vérifie les résultats des tests d'intégration |
| `install` | Installe le livrable dans le dépôt **local** (`~/.m2`), utilisable par d'autres projets locaux |
| `deploy` | Publie le livrable vers un dépôt **distant**, partagé par l'équipe/l'organisation |

**Commandes utiles :**

```bash
mvn clean            # supprime les fichiers générés (dossier target/)
mvn compile           # compile seulement
mvn test              # compile + exécute les tests
mvn package            # compile + teste + empaquette
mvn dependency:tree    # affiche l'arbre complet des dépendances
```

> 💡 Exécuter une phase déclenche **automatiquement** toutes les phases qui la précèdent — `mvn
> package` exécute donc aussi `validate`, `compile` et `test` avant d'empaqueter.

<details>
<summary>🤔 Testez-vous</summary>

Si `mvn test` échoue à cause d'un test qui échoue, `mvn package` va-t-il quand même produire un
`.jar` ?

**Réponse** : non — par défaut, si la phase `test` échoue, Maven **arrête tout le build** et ne
progresse pas jusqu'à `package`. C'est volontaire : livrer un `.jar` dont on sait qu'un test échoue
serait risqué.
</details>

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Lancez `mvn clean test` sur votre projet et repérez, dans la sortie console, le rapport Surefire
(généralement dans `target/surefire-reports/`). Combien de tests ont été exécutés ? Y a-t-il des
échecs ?
{{% /notice %}}

---

## 📚 Références

- Sonatype (2008), *Maven: The Definitive Guide*, O'Reilly Media — référence officielle du plan de
  cours pour Maven.
- Documentation officielle Apache Maven — [maven.apache.org](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html),
  cycle de vie du build.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Maven]({{% relref "/semaine-3/exercice-maven" %}}) pour manipuler
`pom.xml` directement dans votre projet.
{{% /notice %}}
