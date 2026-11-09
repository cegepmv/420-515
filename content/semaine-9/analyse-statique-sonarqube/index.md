+++
title = "Analyse statique avec SonarQube / SonarCloud"
weight = 1
draft = false
+++

Tout ce qui suit est fondé sur la documentation officielle **SonarQube Cloud** (la version hébergée
et gratuite pour les projets open source de SonarSource) — voir
[docs.sonarsource.com/sonarqube-cloud](https://docs.sonarsource.com/sonarqube-cloud/) — ainsi que
sur la configuration réelle du projet `spring-petclinic-rest` utilisé dans ce cours.

---

## 1️⃣ Qu'est-ce que l'analyse statique ?

Jusqu'ici, tout ce que vous avez automatisé pour évaluer la qualité du code **exécutait** le code
(les tests, semaine 4). L'analyse statique fait l'inverse : elle **lit** le code source — sans le
lancer — à la recherche de motifs connus pour être problématiques (complexité excessive,
duplication, mauvaise gestion des ressources, failles de sécurité classiques, etc.).

SonarQube analyse le code à l'aide de **règles**. La documentation officielle les décrit ainsi :

> « Dans SonarQube Cloud, les analyseurs fournissent des règles qui sont exécutées sur le code
> source pour générer des *issues*. [...] Chaque règle est associée à un **type** : bug,
> vulnérabilité, *code smell*, ou *security hotspot*. »
> — [SonarQube rules](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-rules/rules)

Chaque violation détectée dans votre code est appelée une **issue** (anomalie) :

> « Les *issues* SonarQube Cloud sont des erreurs potentielles trouvées dans votre base de code.
> [...] Une analyse détecte une *issue* comme un problème dans votre code. Lorsqu'une règle de
> codage est enfreinte, une *issue* est soulevée. »
> — [Issues — Introduction](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/issues/introduction)

---

## 2️⃣ Les trois qualités logicielles évaluées

SonarQube ne se contente pas de compter des anomalies : chaque règle est rattachée à l'une des
trois **qualités logicielles** suivantes, selon la documentation officielle :

> « Un code de haute qualité contribue à un logiciel sécurisé, fiable et maintenable. [...]
> - **Sécurité** est la protection de votre logiciel contre un accès, un usage ou une destruction
>   non autorisés.
> - **Fiabilité** est une mesure de la capacité de votre logiciel à maintenir son niveau de
>   performance dans des conditions données, pour une période donnée.
> - **Maintenabilité** fait référence à la facilité avec laquelle vous pouvez réparer, améliorer et
>   comprendre le code. »
> — [SonarQube rules — Software qualities](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-rules/rules)

| Qualité | Question posée par la règle | Type historique associé |
|---|---|---|
| **Sécurité** | Ce code pourrait-il être exploité par un attaquant ? | Vulnérabilité |
| **Fiabilité** | Ce code est-il manifestement erroné, ou plus probablement erroné que non ? | Bug |
| **Maintenabilité** | Ce n'est ni un bug, ni une vulnérabilité, mais... | *Code smell* |

**Exemples concrets, un par qualité logicielle** (le genre de code que SonarQube signalerait
réellement dans un projet Java comme celui du cours) :

```java
// 🔴 Sécurité — injection SQL (requête construite par concaténation d'une entrée utilisateur)
String query = "SELECT * FROM owners WHERE id = " + request.getParameter("ownerId");
statement.executeQuery(query);
```

```java
// 🟠 Fiabilité — division par une variable jamais vérifiée (NullPointerException/ArithmeticException possible)
public double tauxReussite(int reussites, int total) {
    return reussites / total; // total == 0 fera planter l'appel, pas de garde
}
```

```java
// 🟡 Maintenabilité — code smell : bloc dupliqué presque à l'identique à trois endroits du projet
if (pet.getType().equals("dog")) {
    log.info("Traitement spécial chien");
    // ... 12 lignes quasi identiques répétées ailleurs dans PetController, VetController...
}
```

Le premier exemple est une **vulnérabilité** exploitable par un attaquant (qualité : sécurité); le
second est un **bug** potentiel, pas malveillant mais incorrect (qualité : fiabilité); le troisième
n'est ni l'un ni l'autre — le code *fonctionne*, mais sera coûteux à maintenir (qualité :
maintenabilité), exactement la distinction faite dans le tableau ci-dessus.

### Sévérités

Chaque qualité logicielle impactée par une règle reçoit une sévérité :

| Sévérité | Définition officielle (traduite) |
|---|---|
| **Blocker** | Probabilité significative de conséquences graves, à corriger immédiatement (ex. plantage en production, faille permettant l'extraction de données sensibles). |
| **High** | Impact élevé sur l'application, à corriger dès que possible. |
| **Medium** | Impact moyen sur l'application. |
| **Low** | Impact faible sur l'application. |

— [SonarQube rules — Severities](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-rules/rules)

**En reprenant les trois exemples ci-dessus** :

- L'injection SQL reçoit typiquement la sévérité **Blocker** en sécurité : une donnée contrôlée
  par l'utilisateur peut directement altérer la requête exécutée contre la base de données — le
  pire des scénarios couverts par la définition officielle.
- La division non protégée reçoit typiquement **High** en fiabilité : l'application plante dans un
  cas précis et reproductible, mais seulement si `total` vaut zéro — un impact réel, mais plus
  circonscrit qu'une faille de sécurité exploitable à distance.
- Le bloc dupliqué reçoit typiquement **Low** en maintenabilité : le code reste fonctionnel
  aujourd'hui, le coût se paie plus tard, au moment où il faudra corriger les trois copies au lieu
  d'une seule.

---

## 3️⃣ Quality Profiles : quelles règles sont réellement actives

Toutes les règles décrites dans les sections précédentes ne sont pas automatiquement appliquées
à votre projet. C'est un **Quality Profile** (profil de qualité) qui détermine, pour chaque
langage, l'ensemble précis de règles activées lors de l'analyse :

> « Un profil de qualité définit l'ensemble des règles de codage à appliquer durant l'analyse.
> [...] Chaque projet a un seul profil de qualité activé, pour chaque langage, à un moment donné.
> Par défaut, le profil de qualité intégré **Sonar way core** est activé pour chaque langage. »
> — [Quality profiles — Introduction](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-quality-profiles/introduction)

SonarQube Cloud fournit trois profils intégrés, chacun avec un niveau de rigueur différent :

> « **Sonar way core** active : toutes les règles de sécurité; les règles de fiabilité
> recommandées de sévérité moyenne ou plus; les règles de maintenabilité recommandées de sévérité
> *Blocker*. [...] **Sonar way extended** [...] ajoute de la rigueur en fiabilité et en
> maintenabilité. [...] **Sonar way comprehensive** active : toutes les règles de sécurité; toutes
> les règles de fiabilité recommandées; toutes les règles de maintenabilité recommandées. »
> — [Built-in quality profiles](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-quality-profiles/built-in-quality-profiles)

| Profil intégré | Rigueur | Cas d'usage typique |
|---|---|---|
| **Sonar way core** (par défaut) | La plus légère | Bon point de départ pour la plupart des projets |
| **Sonar way extended** | Intermédiaire | Équipe déjà à l'aise avec SonarQube, qui veut plus de rigueur |
| **Sonar way comprehensive** | Maximale | Équipe expérimentée, ou évaluation complète des capacités de l'outil |

**Exemple concret** : le projet `spring-petclinic-rest` du cours est analysé en Java. Par défaut,
c'est donc le profil **Sonar way core (Java)** qui est actif — visible dans SonarCloud en ouvrant
le projet, soit sous le panneau **Project Information**, soit sous le menu **Policies** →
**Quality Profiles** (profil assigné par langue). Si une équipe bascule plutôt vers **Sonar way
extended**,
des règles de fiabilité auparavant ignorées (sévérité *Low*) deviennent actives : du code qui
passait inaperçu jusque-là peut soudainement générer de nouvelles *issues*, et potentiellement
faire échouer un quality gate qui passait auparavant — sans qu'une seule ligne de code n'ait
changé.

---

## 4️⃣ Dette technique et notation de maintenabilité

Chaque *code smell* détecté n'a pas le même coût de correction. SonarQube chiffre ce coût sous
forme de **dette technique** :

> « La [dette technique](https://www.sonarsource.com/learn/technical-debt/) est la somme des coûts
> de remédiation des *issues* de maintenabilité. Le coût de remédiation d'une *issue* est l'effort
> (en minutes) évalué pour corriger l'*issue*. [...] Une journée de 8 heures est présumée lorsque la
> dette technique est affichée en jours. »
> — [Understanding measures and metrics — Technical debt](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)

Ce chiffre brut (en minutes ou en jours) devient plus parlant une fois rapporté à la taille du
projet, sous forme de **ratio de dette technique** :

> « Le ratio de dette technique est le rapport entre le coût de développement du logiciel et la
> dette technique (le coût pour la corriger). [...] `sqale_debt_ratio` = dette technique / (coût de
> développement d'une ligne de code × nombre de lignes de code), où le coût de développement d'une
> ligne de code est prédéfini (par défaut, 30 minutes). »
> — [Understanding measures and metrics — Technical debt ratio](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)

**Exemple concret donné par la documentation officielle** : pour un projet avec une dette
technique de **122 563 minutes** et **63 987 lignes de code**, le ratio est de
`122 563 / (30 × 63 987) = 6,4 %`. Selon le barème ci-dessous, ce projet obtiendrait une
notation de maintenabilité **B**.

Ce ratio est ensuite traduit en une note de **A à E**, affichée directement sur votre projet :

> « La notation de maintenabilité reflète la densité de dette technique relative à la taille du
> code. Un ratio plus bas donne une meilleure note. [...] **A** ≤ 5 % · **B** ≥ 5 % à < 10 % ·
> **C** ≥ 10 % à < 20 % · **D** ≥ 20 % à < 50 % · **E** ≥ 50 %. »
> — [Understanding measures and metrics — Maintainability rating](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)

> ⚠️ Une note **globale** flatteuse peut masquer un problème récent : exactement comme pour la
> couverture JaCoCo (section 9️⃣), SonarQube calcule **deux** valeurs distinctes pour la plupart de
> ses métriques — une pour le code global, une pour le « nouveau code » — précisément parce qu'un
> projet legacy avec une dette ancienne importante ne doit pas servir d'excuse pour accepter de la
> **nouvelle** dette.

**Où trouver ces chiffres dans SonarCloud** :

1. Ouvrez votre projet → onglet **Overview** : la note A-E de maintenabilité et le ratio y sont
   déjà affichés en résumé.
2. Pour le détail chiffré : onglet **Reporting** → **Measures** → domaine **Maintainability** →
   vous y trouverez la **Technical Debt** (en minutes/jours), le **Technical Debt Ratio** (le %)
   et la **Maintainability Rating** (la note A-E).
3. Pour le détail *issue par issue* qui compose cette dette : onglet **Issues**, filtré par type
   `Code Smell`.

---

## 5️⃣ Le quality gate et « Clean as You Code »

La **quality gate** (porte de qualité) est l'indicateur central de SonarCloud :

> « Une quality gate SonarQube Cloud est un indicateur qui vous indique si votre code atteint le
> niveau minimal de qualité requis pour votre projet. Elle consiste en un ensemble de conditions
> appliquées aux résultats de chaque analyse. Si les résultats de l'analyse atteignent ou dépassent
> les conditions de la quality gate, un statut **Passed** s'affiche; sinon, un statut **Failed**
> s'affiche. »
> — [Quality gates](https://docs.sonarsource.com/sonarqube-cloud/standards/quality-gates)

Le rôle concret d'une quality gate, selon la documentation, dépend du contexte :

> « Pour la branche principale et les autres branches de longue durée, la quality gate répond à la
> question : *Puis-je livrer mon code aujourd'hui ?* Pour les Pull Requests (et branches de courte
> durée), la quality gate répond à la question : *Puis-je fusionner cette Pull Request ?* »
> — [Quality gates — How quality gates are used](https://docs.sonarsource.com/sonarqube-cloud/standards/quality-gates)

> ⚠️ Une quality gate n'est **pas calculée** après une toute première analyse — il en faut au
> moins deux pour établir une comparaison avec le « nouveau code ». C'est normal de voir
> **Not computed** lors de la toute première exécution sur un projet.

### Le nouveau code : la logique derrière « Clean as You Code »

SonarSource résume sa philosophie par le slogan **« Clean as You Code »** : plutôt que d'exiger
une remise à niveau immédiate de tout un projet legacy, on se concentre sur le code **qu'on
touche aujourd'hui**. Concrètement, SonarQube calcule la plupart de ses métriques deux fois :

> « Pour la plupart des métriques, SonarQube calcule deux valeurs ou notations : une pour le code
> global et une pour le nouveau code. Le code global est tout le code analysé du projet ou de la
> branche, l'ancien code plus le nouveau code. Le nouveau code est le code qui correspond à votre
> définition du nouveau code. C'est le code qui a été récemment ajouté ou modifié. [...] Les
> valeurs calculées peuvent différer entre le code global et le nouveau code parce que la portée
> est différente. Par exemple : votre code global peut obtenir un A en notation de
> maintenabilité, tandis que le nouveau code introduit plus de dette technique et obtient un B. »
> — [Understanding measures and metrics](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)

Cette « définition du nouveau code » n'est pas figée : la documentation propose plusieurs façons
de la configurer par projet — par exemple *Number of days* (tout code modifié dans les X derniers
jours, 30 jours par défaut) ou *Previous version* (tout code modifié depuis le dernier
incrément de version) :

> « Tout code qui a changé dans les X derniers jours est considéré comme du nouveau code. [...] La
> valeur par défaut est 30 jours, 7 ou 14 jours sont d'autres valeurs courantes. »
> — [Quality standards and new code](https://docs.sonarsource.com/sonarqube-cloud/standards/about-new-code)

C'est précisément ce mécanisme qui permet à la quality gate par défaut de **ne cibler que le
nouveau code** : un projet legacy truffé de dette technique ancienne peut donc tout de même
obtenir un quality gate **Passed** sur ses dernières Pull Requests, tant que le code qui y est
ajouté respecte, lui, les standards de qualité — l'inverse (projet globalement propre mais
nouveau code bâclé) provoque, lui, un échec, comme illustré dans l'encadré « Testez-vous » plus
bas.

---

## 6️⃣ Configurer SonarCloud pour votre propre projet

### Créer un compte et une organisation

SonarCloud utilise une **connexion GitHub** plutôt qu'un mot de passe distinct, et structure tout
son fonctionnement autour d'une **organisation** liée à votre compte ou à votre organisation GitHub :

> « Chaque organisation SonarQube Cloud correspond un-à-un à une organisation GitHub ou à un compte
> personnel. »
> — [docs.sonarsource.com — Getting started with GitHub](https://docs.sonarsource.com/sonarqube-cloud/getting-started/github.md)

1. Allez sur [sonarcloud.io](https://sonarcloud.io) et connectez-vous **avec GitHub**.
2. Installez l'application GitHub SonarQube Cloud sur votre compte ou votre organisation GitHub
   (celle du dépôt de votre équipe).
3. Choisissez un plan, puis sélectionnez **Create organization** — l'organisation SonarCloud est
   créée, liée un-à-un à votre compte/organisation GitHub.
4. Importez votre dépôt comme projet (un dépôt = un projet SonarCloud).

> ⚠️ **Erreur fréquente** : à l'import, SonarCloud active par défaut l'**Automatic Analysis**
> (analyse automatique depuis GitHub, sans passer par votre pipeline CI). Cette méthode est
> **incompatible** avec l'analyse CI décrite plus bas (section 7️⃣) : si les deux sont actives en
> même temps, l'exécution Maven échoue avec `You are running CI analysis while Automatic Analysis
> is enabled`. Désactivez-la dès l'import : ouvrez le **projet** (pas l'organisation) → onglet
> **Administration** → **Analysis Method** → décochez **Automatic Analysis**.

### Choisir le bon plan : Free vs OSS

Deux plans gratuits existent, avec une différence importante pour une équipe de plusieurs
personnes :

| Plan | Membres max | Dépôts privés |
|---|---|---|
| **Free** | **5** | jusqu'à 50k lignes de code |
| **OSS** (*open source*) | **illimité** | aucun |

> « Nombre maximal de membres de l'organisation : 5 (Free) [...] Illimité (Team/Enterprise). »
> — [SonarQube Cloud — Subscription plans](https://docs.sonarsource.com/sonarqube-cloud/administering-sonarcloud/managing-subscription/subscription-plans.md)

Pour une équipe du cours (dépôts **publics**, plus de 5 personnes possibles sur une même
organisation GitHub) : sélectionnez **« Get SonarQube for OSS »** au moment de la création de
l'organisation — ce choix n'est **pas automatique** du simple fait que les dépôts soient publics,
il doit être fait explicitement à cette étape. Une organisation déjà créée en plan Free doit être
migrée manuellement vers OSS; rien ne bascule d'elle-même.

### Générer un jeton d'analyse (`SONAR_TOKEN`)

1. Menu du compte (en haut à droite) → **My account** → onglet **Access Tokens**.
2. Sous **Generate Tokens** : donnez un nom (ex. `github-actions-ci`) et choisissez une durée
   d'expiration.
3. Cliquez **Generate Token** — la valeur s'affiche **une seule fois** :

> « Copiez-le immédiatement en sélectionnant l'outil de copie devant la valeur. [...] Les jetons
> sans date d'expiration sont automatiquement supprimés après 60 jours d'inactivité. »
> — [SonarQube Cloud — Managing personal access tokens](https://docs.sonarsource.com/sonarqube-cloud/managing-your-account/managing-tokens.md)

### Ajouter le jeton comme secret GitHub

Dans le dépôt GitHub (pas dans le code, jamais commité) : `Settings` → `Secrets and variables` →
`Actions` → `New repository secret` → nom `SONAR_TOKEN`, valeur collée depuis l'étape précédente.
Un **repository secret** suffit ici (plutôt qu'un *environment secret*) : l'analyse SonarCloud
tourne sur chaque Pull Request et chaque push, sans notion de déploiement vers un environnement
particulier — contrairement à un déploiement Kubernetes en production (semaines 11-12), où un
environnement avec approbation manuelle aura davantage de sens.

> ⚠️ Ce jeton est un **jeton personnel**, lié au compte qui l'a généré : si cette personne quitte
> l'équipe ou supprime son compte, le jeton est révoqué et le pipeline cesse de fonctionner. À
> partir du plan Team, SonarCloud propose des *Scoped Organization Tokens*, non liés à une
> personne précise — non disponibles sur les plans Free/OSS utilisés dans ce cours.

---

## 7️⃣ Intégration réelle au projet du cours

Le dépôt upstream `spring-petclinic-rest` déclenche une analyse SonarCloud directement depuis
Maven, dans son pipeline GitHub Actions (vu en référence à la semaine 5, déclenchement manuel dans
le dépôt du cours) :

```yaml
# .github/workflows/maven-build-master.yml (projet spring-petclinic-rest)
- name: Build and analyze
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: ./mvnw -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=spring-petclinic_spring-petclinic-rest -Dsonar.organization=spring-petclinic
```

Deux éléments essentiels dans cette commande :

- Le **plugin Maven du scanner SonarQube** (`sonar-maven-plugin:sonar`) s'ajoute directement après
  `verify` — l'analyse ne remplace pas le cycle de vie Maven habituel (semaine 3), elle s'y greffe.
- Un **jeton d'authentification** (`SONAR_TOKEN`) est requis en plus du `GITHUB_TOKEN` habituel —
  ce jeton est généré depuis SonarCloud et stocké comme secret du dépôt GitHub, exactement comme
  vous l'avez fait pour d'autres secrets à la semaine 5.

Le `pom.xml` du même projet déclare les propriétés correspondantes :

```xml
<!-- SonarQube configuration -->
<sonar.organization>spring-petclinic-rest</sonar.organization>
<sonar.host.url>https://sonarcloud.io</sonar.host.url>
```

> ⚠️ Notez que la commande du workflow utilise `-Dsonar.organization=spring-petclinic` (sans
> `-rest`) tandis que le `pom.xml` déclare `spring-petclinic-rest`. Un paramètre passé en ligne de
> commande (`-D...`) **écrase** toujours la valeur équivalente définie dans le `pom.xml` — c'est la
> valeur de la commande qui est réellement utilisée à l'exécution. Ce genre d'écart, laissé sans
> conséquence ici parce qu'un seul des deux est actif, illustre bien pourquoi une configuration
> dupliquée à deux endroits est elle-même un *code smell* à surveiller.

### Exemple : intégrer `verify` + SonarCloud dans un pipeline Docker existant

Un cas fréquent dans les projets d'équipe de ce cours : un workflow qui construisait déjà une image
Docker avec `mvn package -DskipTests` (tests ignorés pour aller vite), auquel on veut ajouter
l'analyse SonarCloud **sans pousser une image non testée**. La solution consiste à ajouter un
**job séparé** qui s'exécute en premier, et à faire dépendre le job de build via `needs:` :

```yaml
jobs:
  test-and-analyze:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
        with:
          # nécessaire pour que l'analyse SonarCloud puisse comparer avec l'historique (New Code)
          fetch-depth: 0

      - name: Set up JDK 25
        uses: actions/setup-java@v5
        with:
          java-version: '25'
          distribution: 'temurin'
          cache: maven

      - name: Build, tests et analyse SonarCloud
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: ./mvnw -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=${{ vars.SONAR_PROJECT_KEY }}

  build-and-push:
    needs: test-and-analyze   # <- n'exécute ce job QUE si test-and-analyze a réussi
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Build with Maven
        run: ./mvnw -B package -DskipTests
      # ... (étapes de build/push Docker)
```

 Points importants :
- **`fetch-depth: 0`** : par défaut, `actions/checkout` ne récupère que le **dernier commit** (clone
  superficiel) pour aller plus vite. SonarCloud a besoin de **tout l'historique Git** pour calculer
  le **blame** (quel auteur/commit a introduit chaque ligne) et le **New Code period** (comparaison
  avec une version de référence, voir section 5️⃣). Sans `fetch-depth: 0`, l'analyse fonctionne
  quand même mais affiche un avertissement *"Shallow clone detected"* et perd l'information
  d'auteur/date sur les *issues*.

> ℹ️ **Pratique par défaut dans l'industrie** : la documentation officielle recommande en fait de
> garder `sonar.organization` **et** `sonar.projectKey` directement dans le `pom.xml` versionné
> pour un projet unique — ce ne sont pas des identifiants secrets, et un paramètre de ligne de
> commande n'est utile que si la valeur **varie réellement selon le pipeline/l'environnement**.
>
> « Pour un seul projet SonarQube Cloud, gardez `sonar.organization` et `sonar.projectKey` dans le
> `pom.xml` versionné. [...] Utilisez des paramètres CI quand ces valeurs varient selon
> l'environnement, le dépôt ou le pipeline. [...] Gardez le jeton Sonar dans votre gestionnaire de
> secrets CI, pas dans le POM. »
> — [SonarQube Cloud — Project analysis](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/project-analysis.md)
>
> Le passage par `vars.SONAR_PROJECT_KEY` ci-dessus reste pertinent dans **ce cours** : chaque
> équipe travaille sur son propre fork avec sa propre organisation/clé SonarCloud, donc la valeur
> varie bel et bien d'un dépôt à l'autre — exactement le cas d'usage que la documentation reconnaît
> comme légitime pour une surcharge CI.

**Plan de secours : rendre un paramètre optionnel sans écraser le `pom.xml`**

Un paramètre `-D...` écrase toujours la valeur du `pom.xml`, même avec une chaîne vide. Si vous
voulez garder une valeur par défaut dans le `pom.xml` (ex. `sonar.organization`) et ne l'écraser
**que** si une variable GitHub existe, il faut construire la commande conditionnellement dans le
`run:` plutôt que d'insérer directement `-Dsonar.organization=${{ vars.SONAR_ORGANIZATION }}` :

```yaml
run: |
  ARGS="-Dsonar.projectKey=${{ vars.SONAR_PROJECT_KEY }}"
  if [ -n "${{ vars.SONAR_ORGANIZATION }}" ]; then
    ARGS="$ARGS -Dsonar.organization=${{ vars.SONAR_ORGANIZATION }}"
  fi
  ./mvnw -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar $ARGS
```

- Si `SONAR_ORGANIZATION` existe : le flag est ajouté → écrase le `pom.xml`.
- Si elle est absente (chaîne vide) : le flag n'est **jamais construit**, donc jamais passé →
  Maven retombe sur `<sonar.organization>` défini dans le `pom.xml`.

### Faire échouer le build si le quality gate échoue

Par défaut, `sonar-maven-plugin:sonar` envoie l'analyse à SonarCloud de façon **asynchrone** et ne
bloque **pas** le build Maven, même si le quality gate échoue — il confirme seulement l'envoi des
résultats. Pour que votre job `test-and-analyze` (et donc `build-and-push`, via `needs:`) échoue
réellement quand le quality gate échoue, ajoutez `-Dsonar.qualitygate.wait=true` :

```bash
./mvnw -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
  -Dsonar.projectKey=... -Dsonar.qualitygate.wait=true
```

Le scanner attend alors le résultat du quality gate (300 secondes par défaut — ajustable avec
`-Dsonar.qualitygate.timeout=<secondes>`) avant de terminer, ce qui **rallonge le pipeline**. C'est
un mécanisme différent — et complémentaire — de la décoration de Pull Request (section 8️⃣) : la
décoration bloque la **fusion** via les status checks GitHub (si une règle de protection de branche
l'exige), alors que `sonar.qualitygate.wait=true` fait échouer le **job CI lui-même**, peu importe
que des règles de protection de branche soient configurées ou non.

---

## 8️⃣ Décoration des Pull Requests

SonarCloud n'attend pas qu'une Pull Request soit fusionnée pour l'analyser. C'est même l'un des
trois points d'intégration mis en avant par la documentation :

> « SonarQube Cloud s'intègre à votre flux de travail à trois endroits : dans l'IDE (via SonarQube
> for IDE), sur la Pull Request, et sur la base de code. »
> — [What SonarQube Cloud can do](https://docs.sonarsource.com/sonarqube-cloud/discovering-sonarcloud/what-sonarcloud-can-do)

Concrètement, l'analyse d'une Pull Request ne recalcule pas le quality gate du projet au complet :

> « L'analyse de Pull Request ne rapporte que les *issues* introduites par la Pull Request
> elle-même. Lorsque SonarQube Cloud analyse une Pull Request **P** pour la fusion de la branche
> **B** dans la branche de base **T**, il scanne le commit HEAD de **B** et compare le résultat
> avec la dernière analyse de **T**. Seules les *issues* qui apparaissent dans **B** mais pas dans
> **T** sont rapportées dans les résultats de l'analyse. »
> — [Pull request analysis](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/pull-request-analysis)

Le résultat (statut du quality gate et résumé des *issues*) ne reste pas confiné à l'interface de
SonarCloud : il est aussi publié directement dans la Pull Request GitHub elle-même, un mécanisme
appelé **décoration de Pull Request** :

> « En plus d'apparaître dans l'interface de SonarQube Cloud, le statut du quality gate et un
> résumé des résultats apparaissent aussi dans l'interface de votre plateforme DevOps (c'est-à-dire
> dans la vue de Pull Request de GitHub, Bitbucket Cloud, Azure DevOps ou GitLab). C'est ce qu'on
> appelle la décoration de Pull Request. Mais ce n'est pas que décoratif ! Elle s'intègre aussi à
> votre plateforme DevOps pour **bloquer la fusion** de la Pull Request si le quality gate
> échoue. »
> — [Pull request analysis — Pull request decoration](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/pull-request-analysis)

> ⚠️ Dans le plan gratuit de SonarQube Cloud, la documentation précise une limite importante pour
> des dépôts publics comme ceux du cours : « l'analyse de Pull Request n'est disponible que
> lorsque la Pull Request est fusionnée dans la branche principale ». La décoration **avant**
> fusion (commentaire en direct sur une PR encore ouverte) nécessite un plan payant — un détail à
> garder en tête si votre analyse n'apparaît pas tout de suite sur une PR encore ouverte.

**Exemple concret** : sur une Pull Request qui ajoute une méthode avec une duplication de code et
une couverture de tests insuffisante, le commentaire automatique déposé par SonarCloud affiche
typiquement :

```
Quality Gate Failed  ❌

2 New issues  •  0 Security Hotspots
Coverage on New Code: 42.0%  (required ≥ 80%)
Duplication on New Code: 15.0%  (required ≤ 3%)
```

Ce résumé (statut du quality gate, puis les métriques clés calculées uniquement sur le **nouveau
code** de la PR) permet à l'équipe de voir, directement dans l'onglet *Files changed* de GitHub,
pourquoi la fusion est bloquée — sans avoir à ouvrir SonarCloud.

Ce mécanisme explique pourquoi la section précédente distinguait deux questions différentes posées
par le quality gate : sur la branche principale, « puis-je livrer mon code aujourd'hui ? »; sur une
Pull Request, « puis-je fusionner cette Pull Request ? ». La décoration de Pull Request est
précisément ce qui rend cette seconde question visible **sans quitter GitHub**.

---

## 9️⃣ Fournir la couverture de tests à SonarCloud : JaCoCo

SonarCloud ne mesure pas lui-même la couverture de tests (semaine 4) : il **lit** un rapport déjà
produit par un autre outil, **JaCoCo**, et l'intègre à son analyse. Le `pom.xml` du projet
configure ce plugin avec deux exécutions Maven distinctes :

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${maven.jacoco-plugin.version}</version>
    <configuration>
        <excludes>
            <!-- Generated code -->
            <exclude>**/org/springframework/samples/petclinic/rest/dto/**</exclude>
            <exclude>**/org/springframework/samples/petclinic/rest/api/**</exclude>
        </excludes>
    </configuration>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.85</minimum>
                            </limit>
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.66</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
        <execution>
            <id>report</id>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

| Exécution | Rôle |
|---|---|
| `prepare-agent` | Instrumente le code **avant** l'exécution des tests, pour observer quelles lignes sont réellement exercées |
| `check` | Échoue le build Maven lui-même si la couverture globale (`BUNDLE`) descend sous **85 % des lignes** ou **66 % des branches** |
| `report` | Génère le rapport au format attendu par SonarCloud, **après** les tests |

> 💡 Les exclusions (`exclude`) retirent volontairement le code **généré automatiquement** (DTO et
> API générés par OpenAPI Generator, voir semaine 8) du calcul de couverture — ce code n'a jamais
> été écrit à la main, il serait absurde d'exiger qu'il soit testé comme du code métier.

Cette double configuration illustre une idée importante : un **quality gate** peut combiner des
métriques de **plusieurs outils** (ici, les règles SonarQube elles-mêmes **et** la couverture
JaCoCo) pour décider si un changement est acceptable.
---

## 📚 Références

- [SonarQube rules](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-rules/rules)
  — types de règles, qualités logicielles, sévérités.
- [Issues — Introduction](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/issues/introduction)
  — cycle de vie d'une *issue*.
- [Understanding measures and metrics](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)
  — dette technique, ratio de dette technique, notation de maintenabilité, code global vs nouveau code.
- [Quality gates](https://docs.sonarsource.com/sonarqube-cloud/standards/quality-gates)
  — définition et usage du quality gate.
- [Quality standards and new code](https://docs.sonarsource.com/sonarqube-cloud/standards/about-new-code)
  — définitions du « nouveau code » (Clean as You Code).
- Projet `spring-petclinic-rest` —
  [`.github/workflows/maven-build-master.yml`](https://github.com/spring-petclinic/spring-petclinic-rest/blob/main/.github/workflows/maven-build-master.yml)
  et [`pom.xml`](https://github.com/spring-petclinic/spring-petclinic-rest/blob/main/pom.xml).
- [SonarQube Cloud — Project analysis](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/project-analysis.md)
  — où déclarer `sonar.organization`/`sonar.projectKey` (pom.xml vs paramètres CI), et le paramètre
  `sonar.qualitygate.wait` pour faire échouer le build sur un quality gate en échec.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Analyse de vulnérabilités avec Trivy]({{% relref "/semaine-9/analyse-vulnerabilites-trivy" %}})
pour voir comment une deuxième famille d'outils détecte un risque que SonarQube ne couvre pas :
les failles **connues** dans vos dépendances et votre image Docker.
{{% /notice %}}
