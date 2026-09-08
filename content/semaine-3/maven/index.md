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

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <groupId>ca.cegepmv.legacy</groupId>
    <artifactId>mon-projet</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

| Élément | Rôle |
|---|---|
| `groupId` | Identifiant de l'organisation/projet (convention : nom de domaine inversé) |
| `artifactId` | Nom du module/projet lui-même |
| `version` | Version du projet — `SNAPSHOT` signifie « en développement, pas figée » |
| `packaging` | Type de livrable produit (`jar`, `war`, `pom` pour un module parent) |
| `properties` | Variables réutilisables ailleurs dans le fichier (ex. version de Java) |
| `dependencies` | Librairies externes requises |
| `build > plugins` | Outils qui interviennent pendant le build (compiler, tester, empaqueter...) |

---

## 2️⃣ Dépendances et *scope*

Le `scope` d'une dépendance détermine **quand** elle est disponible :

| Scope | Disponible pour... | Exemple typique |
|---|---|---|
| `compile` *(défaut)* | Compilation ET exécution ET tests | Une librairie utilisée partout (ex. Spring) |
| `test` | Seulement la compilation/exécution des tests | JUnit, Mockito |
| `provided` | Compilation, mais fournie par l'environnement d'exécution (pas empaquetée) | API Servlet sur un serveur qui la fournit déjà |
| `runtime` | Exécution seulement, pas nécessaire pour compiler | Un pilote JDBC |

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

```
[INFO] ca.cegepmv.legacy:mon-projet:jar:1.0.0-SNAPSHOT
[INFO] +- org.springframework.boot:spring-boot-starter-web:jar:3.3.0:compile
[INFO] |  \- com.fasterxml.jackson.core:jackson-databind:jar:2.17.1:compile
[INFO] \- com.mabibliotheque:validation-utils:jar:2.0:compile
[INFO]    \- com.fasterxml.jackson.core:jackson-databind:jar:2.15.0:compile (omitted for conflict)
```

Ici, `jackson-databind` est demandé en **deux versions différentes** (2.17.1 et 2.15.0) par deux
chemins différents — Maven a choisi 2.17.1 (la plus proche) et **ignoré** 2.15.0
(`omitted for conflict`). `mvn dependency:tree` est l'outil de diagnostic à réflexe immédiat
devant tout comportement inattendu lié à une librairie.

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

## 5️⃣ Déboguer des tests exécutés par Maven

Poser un breakpoint dans un test et lancer `mvn test` en ligne de commande **ne s'arrête jamais**
sur ce breakpoint — même si le même test, lancé directement depuis l'IDE, fonctionne très bien
avec le débogueur. Ce n'est pas un bogue de l'IDE : c'est le fonctionnement normal de Maven.

### Pourquoi ça ne s'arrête pas

Le plugin `maven-surefire-plugin` exécute les tests dans un **processus Java séparé** (un *fork*),
distinct du processus Maven lui-même. Un breakpoint posé dans votre IDE ne surveille que les
processus auxquels l'IDE est **attaché** — pas ce fork externe créé par Maven, qui roule sans lien
avec l'IDE.

### Option 1 (la plus simple) : lancer le test depuis l'IDE, pas via `mvn`

Si le but est de déboguer **un test précis**, la façon la plus directe est de ne pas passer par
Maven du tout : clic droit sur la méthode de test → **Debug** (IntelliJ) ou l'icône *Debug Test*
au-dessus de la méthode (VS Code, extension *Test Runner for Java*). L'IDE lance alors le test
directement dans un processus qu'il contrôle, et les breakpoints fonctionnent normalement.

> ✅ Suffisant dans la grande majorité des cas — à essayer **avant** les options suivantes.

### Option 2 : attacher le débogueur au processus lancé par Maven

Utile si le test doit **impérativement** être lancé via `mvn` (ex. pour reproduire un comportement
qui dépend d'un profil Maven précis, ou d'options passées uniquement en ligne de commande).

#### Option 2a (la plus simple, directement dans IntelliJ) : le panneau Maven

1. Ouvrez le panneau **Maven** (onglet vertical à droite de la fenêtre, ou **View → Tool Windows
   → Maven** s'il n'est pas visible).
2. Cliquez sur l'icône **Execute Maven Goal** en haut du panneau.
3. Tapez la commande, en ciblant au besoin un test précis :
   ```
   test -Dtest=NomDeLaClasse#nomDeLaMéthode
   ```
4. Le popup propose deux boutons : **Run** et **Debug** (🐞) — cliquez sur **Debug**.
5. IntelliJ gère lui-même le fork et l'attache **automatiquement** au débogueur — aucun port à
   configurer manuellement, aucune configuration *Remote JVM Debug* à créer.

> ✅ À essayer en premier si le panneau Maven est disponible — évite toute la mécanique manuelle
> ci-dessous.

#### Option 2b : manuellement, via un terminal (utile hors IntelliJ, ou si le panneau Maven n'aide pas)

1. Démarrez Maven en mode debug — le processus se lance puis **attend** qu'un débogueur s'y
   connecte avant de continuer. Si votre projet utilise le **Maven Wrapper** (fichier `mvnw`/
   `mvnw.cmd` à la racine — c'est le cas le plus fréquent), la commande `mvnDebug` n'est
   généralement **pas disponible** (elle vient d'une installation Maven autonome, pas du wrapper) :
   utilisez plutôt le flag équivalent avec `mvnw` :
   ```powershell
   .\mvnw.cmd test "-Dmaven.surefire.debug" "-Dtest=NomDeLaClasse#nomDeLaMéthode"
   ```
   (si Maven est installé de façon autonome sur votre poste et accessible en ligne de commande, la
   forme courte `mvnDebug test` fait la même chose.)
2. La console affiche un message du genre :
   ```
   Listening for transport dt_socket at address: 8000
   ```
3. Attachez le débogueur au processus en attente — deux façons dans IntelliJ :
   - **Attach to Process** (le plus simple, aucune configuration à créer) : menu **Run → Attach to
     Process…** → repérez dans la liste le processus contenant `ForkedBooter` ou `surefire` →
     cliquez dessus.
   - **Remote JVM Debug** (si vous voulez réutiliser la même configuration plus tard) : menu
     **Run → Edit Configurations…** → clic droit dans le panneau de gauche → **Add New
     Configuration** (ou **Alt+Insert**) → **Remote JVM Debug** → port `8000` (ou celui affiché) →
     **OK**, puis sélectionnez cette configuration et cliquez sur 🐞.
   - **VS Code** (extension Java) : `.vscode/launch.json` →
     ```json
     {
       "type": "java",
       "name": "Attacher à Maven",
       "request": "attach",
       "hostName": "localhost",
       "port": 8000
     }
     ```
     puis lancer cette configuration depuis l'onglet *Run and Debug*.
4. Une fois attaché, le processus Maven reprend son exécution et **s'arrête normalement** sur vos
   breakpoints.

<details>
<summary>🤔 Testez-vous</summary>

Vous posez un breakpoint dans une méthode de test, puis lancez `mvn test` normalement (sans mode
debug) depuis un terminal. Le programme s'exécute jusqu'au bout sans jamais s'arrêter. Pourquoi, et
que devriez-vous faire différemment ?

**Réponse** : `mvn test` exécute les tests dans un processus Java **séparé** (fork Surefire) auquel
aucun débogueur n'est attaché — le breakpoint est ignoré, pas « manqué ». Il faut soit lancer le
test directement depuis l'IDE (option 1), soit passer par le panneau Maven en mode Debug (option
2a), soit démarrer Maven avec `-Dmaven.surefire.debug` et attacher un débogueur distant sur le
port indiqué (option 2b).
</details>

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
