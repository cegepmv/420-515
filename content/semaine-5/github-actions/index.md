+++
title = "GitHub Actions — les bases"
weight = 3
draft = false
+++

Avant de construire votre propre pipeline (voir [Intégration continue]({{% relref
"/semaine-5/integration-continue" %}})), il est utile de comprendre le vocabulaire et la mécanique
de **GitHub Actions** en profondeur : ce n'est pas un outil réservé à la CI, mais un moteur
d'automatisation générique déclenché par des événements sur un dépôt GitHub.

---

## 1️⃣ Vocabulaire de base

| Terme | Définition |
|---|---|
| **Workflow** | Un fichier YAML (`.github/workflows/*.yml`) décrivant une automatisation complète, déclenchée par un ou plusieurs événements |
| **Événement** (*event*) | Ce qui déclenche l'exécution d'un workflow (`push`, `pull_request`, `workflow_dispatch`, `schedule`, etc.) |
| **Job** | Un ensemble d'étapes exécutées séquentiellement, sur une seule machine virtuelle (*runner*) |
| **Step** | Une étape individuelle à l'intérieur d'un job : soit une commande shell (`run:`), soit une **action** réutilisable (`uses:`) |
| **Action** | Une unité de code réutilisable et partageable (souvent publiée sur le [Marketplace](https://github.com/marketplace?type=actions)), ex. `actions/checkout@v5` |
| **Runner** | La machine virtuelle éphémère (neuve à chaque exécution) sur laquelle un job s'exécute (`ubuntu-latest`, `windows-latest`, etc., ou un runner auto-hébergé) |
| **Run** (exécution) | Une instance concrète d'exécution d'un workflow, déclenchée par un événement précis (visible sous l'onglet **Actions** du dépôt) |

> 💡 Un dépôt peut contenir **plusieurs** fichiers de workflow dans `.github/workflows/`, chacun
> avec ses propres déclencheurs et responsabilités (ex. un pour la CI, un autre pour publier une
> documentation, un autre pour créer une release).

---

## 2️⃣ Anatomie annotée d'un workflow

```yaml
# .github/workflows/ci.yml
name: CI                              # (1) Nom affiché dans l'onglet Actions

on:                                    # (2) Événements déclencheurs
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:                                  # (3) Un ou plusieurs jobs
  build-and-test:                     # Identifiant du job (utilisé par `needs:` au besoin)
    runs-on: ubuntu-latest             # (4) Le runner sur lequel ce job s'exécute

    steps:                             # (5) Étapes exécutées dans l'ordre
      - uses: actions/checkout@v5      # (6) Une action réutilisable...
      - name: Set up JDK 25
        uses: actions/setup-java@v5
        with:                          # ...paramétrée par des entrées (`with:`)
          java-version: '25'
          distribution: 'temurin'
      - name: Build and run tests
        run: ./mvnw -B clean verify    # (7) ...ou une commande shell directe (`run:`)
```

1. **`name`** — purement cosmétique, identifie le workflow dans l'interface GitHub.
2. **`on`** — un ou plusieurs événements. Ici, le workflow se déclenche à chaque `push` sur `main`
   **et** à chaque Pull Request ciblant `main` (voir la section suivante pour d'autres événements).
3. **`jobs`** — un dictionnaire de jobs; par défaut, des jobs différents s'exécutent **en
   parallèle**, sur des runners distincts et indépendants (sauf dépendance explicite via `needs:`).
4. **`runs-on`** — chaque job démarre sur une machine virtuelle **neuve**, sans aucun état laissé
   par une exécution précédente : rien n'est présumé installé sauf ce que le job installe lui-même.
5. **`steps`** — exécutées **séquentiellement**, dans l'ordre d'écriture, à l'intérieur d'un même
   job. Si une étape échoue, les étapes suivantes du job sont annulées par défaut.
6. **`uses:`** — référence une action existante, ici `actions/checkout@v5` (récupère le code du
   commit courant — sans cette étape, le runner est une machine vide qui ne connaît même pas le
   contenu du dépôt).
7. **`run:`** — exécute directement une commande shell sur le runner, comme dans un terminal.

---

## 3️⃣ Événements déclencheurs courants

| Événement | Se déclenche quand… |
|---|---|
| `push` | Un commit est poussé sur une branche (filtrable par `branches:`) |
| `pull_request` | Une Pull Request est ouverte, mise à jour, ou rouverte vers une branche ciblée |
| `workflow_dispatch` | Quelqu'un déclenche le workflow **manuellement** depuis l'onglet Actions |
| `schedule` | À intervalle régulier, selon une syntaxe cron (ex. tous les jours à minuit) |

```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:      # bouton "Run workflow" disponible manuellement, en plus des déclencheurs automatiques
```

> ⚠️ Le dépôt du cours contient des workflows hérités du projet upstream configurés en
> `workflow_dispatch` uniquement (voir [Intégration continue]({{% relref "/semaine-5/integration-continue" %}})) — ils ne se déclenchent **jamais** automatiquement sur
> `push`/`pull_request`, volontairement, pour ne pas faire échouer votre CI dès le TP1.

---

## 4️⃣ Jobs : parallélisme, dépendances et matrices

Par défaut, plusieurs jobs d'un même workflow s'exécutent **en parallèle** :

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [ ... ]

  lint:
    runs-on: ubuntu-latest
    steps: [ ... ]
```

`build` et `lint` démarrent en même temps, sur deux runners distincts. Pour forcer un ordre (ex.
n'exécuter les tests qu'après un build réussi), on utilise `needs:` :

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [ ... ]

  deploy:
    needs: build          # attend que `build` se termine avec succès avant de démarrer
    runs-on: ubuntu-latest
    steps: [ ... ]
```

Une **matrice** (`strategy.matrix`) permet d'exécuter le **même** job plusieurs fois, avec des
paramètres différents (ex. plusieurs versions de JDK) :

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java: [ '21', '25' ]
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-java@v5
        with:
          java-version: ${{ matrix.java }}
          distribution: 'temurin'
      - run: ./mvnw -B clean verify
```

Ce job s'exécute **deux fois en parallèle** (une fois par version de JDK dans la matrice), sur deux
runners distincts — utile pour confirmer qu'un projet reste compatible avec plusieurs versions.

---

## 5️⃣ Variables, secrets et artefacts

### Variables d'environnement et expressions `${{ }}`

Les doubles accolades `${{ }}` évaluent une **expression** GitHub Actions (contexte, fonction,
variable) :

```yaml
- name: Afficher la branche courante
  run: echo "Branche : ${{ github.ref_name }}"
```

### Secrets

Un **secret** (configuré dans les paramètres du dépôt : *Settings → Secrets and variables →
Actions*) est une valeur sensible (jeton d'API, mot de passe) jamais affichée en clair dans les
journaux :

```yaml
- name: Build
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}   # secret prédéfini, fourni automatiquement par GitHub
  run: ./mvnw -B verify
```

> ⚠️ Ne mettez **jamais** de secret directement en clair dans un fichier de workflow (il serait
> visible par quiconque a accès au dépôt, y compris dans l'historique Git) — passez toujours par un
> secret configuré côté GitHub.

### Artefacts

Un **artefact** est un fichier (ou dossier) produit par un job et conservé après l'exécution (ex.
un rapport de tests ou un `.jar` compilé), téléchargeable depuis l'onglet Actions :

```yaml
- name: Publier le rapport de tests
  uses: actions/upload-artifact@v5
  with:
    name: rapport-surefire
    path: target/surefire-reports/
```

Le même mécanisme permet de conserver le **`.jar`** produit par `./mvnw -B clean package` (ou
`verify`), pour le récupérer sans avoir à reconstruire le projet — utile pour l'inspecter, le
déployer manuellement, ou le réutiliser dans un job suivant :

```yaml
- name: Build
  run: ./mvnw -B clean package -DskipTests   # produit le .jar dans target/

- name: Publier le .jar en artefact
  uses: actions/upload-artifact@v5
  with:
    name: application-jar
    path: target/*.jar          # ex. target/spring-petclinic-rest-*.jar
```

Le `.jar` publié apparaît ensuite dans la section **Artifacts** de l'exécution du workflow (onglet
**Actions**), téléchargeable manuellement pendant une durée limitée (90 jours par défaut).

> 💡 `upload-artifact` ne **déploie** rien : il ne fait que conserver un fichier produit par le job
> pour le rendre téléchargeable depuis l'interface GitHub. Publier une vraie release (ex. sur
> GitHub Releases ou un registre d'artefacts Maven) est une étape distincte, hors de la portée de
> cette introduction.

### Cache Maven

Chaque runner GitHub Actions démarre **neuf** : sans cache, Maven retélécharge la totalité du
dépôt local (`~/.m2/repository`, toutes les dépendances du projet) à **chaque** exécution du
pipeline — plusieurs dizaines de secondes, voire minutes, perdues à chaque `push`.

L'option `cache: maven` de `actions/setup-java` active automatiquement ce cache, sans configuration
additionnelle :

```yaml
- name: Set up JDK 25
  uses: actions/setup-java@v5
  with:
    java-version: '25'
    distribution: 'temurin'
    cache: maven          # met en cache ~/.m2/repository entre les exécutions
```

Concrètement :

1. Au premier run, `setup-java` télécharge toutes les dépendances Maven normalement, puis
   **sauvegarde** le contenu de `~/.m2/repository` comme cache, associé à une clé calculée à partir
   du contenu du fichier `pom.xml` (via un hash).
2. Aux runs suivants, si le `pom.xml` **n'a pas changé**, la clé de cache correspond déjà à un
   cache existant : `setup-java` **restaure** directement `~/.m2/repository` plutôt que de tout
   retélécharger — le pipeline gagne un temps considérable.
3. Si le `pom.xml` change (ex. une dépendance ajoutée ou une version modifiée), la clé de cache
   change aussi : un nouveau cache est reconstruit automatiquement, sans intervention manuelle.

> 💡 Ce cache ne concerne que les **dépendances téléchargées** (`~/.m2/repository`) — il n'a aucun
> effet sur la durée de compilation ou d'exécution des tests eux-mêmes, seulement sur la partie
> « téléchargement » du build.

<details>
<summary>🤔 Testez-vous</summary>

Vous ajoutez une nouvelle dépendance dans le `pom.xml` de votre projet. Le prochain pipeline
CI semble « ignorer » le cache Maven habituel et prend plus de temps que d'habitude à s'exécuter.
Est-ce normal, et pourquoi ?

**Réponse** : oui, c'est le comportement attendu. La clé de cache est calculée à partir du contenu
du `pom.xml` : dès que celui-ci change, l'ancienne clé ne correspond plus, donc l'ancien cache
n'est pas restauré. `setup-java` télécharge alors les dépendances normalement (incluant la
nouvelle), puis sauvegarde un **nouveau** cache associé à la nouvelle clé — les prochains runs avec
ce `pom.xml` inchangé profiteront à nouveau du cache.
</details>

---

<details>
<summary>🤔 Testez-vous</summary>

Un workflow contient deux jobs, `tests` et `publish`, sans clause `needs:` entre eux. Le job
`publish` échoue de façon intermittente parce qu'il tente parfois de publier un artefact que
`tests` n'a pas encore fini de produire. Quelle est la cause, et comment la corriger ?

**Réponse** : sans `needs:`, GitHub Actions exécute les jobs **en parallèle** par défaut — rien ne
garantit que `tests` se termine avant `publish`. Il faut ajouter `needs: tests` au job `publish`
pour forcer l'ordre d'exécution et garantir que `tests` (et la production de l'artefact) est
terminé avant que `publish` démarre.
</details>

---

## 6️⃣ Conditions d'exécution : `if`, `success()`, `failure()`, `always()`, `cancelled()`

Par défaut, une **étape** (step) ne s'exécute que si toutes les étapes précédentes du même job ont
réussi, et un **job** ne s'exécute que si tous les jobs listés dans `needs:` ont réussi. La clause
`if:` permet de personnaliser cette règle, souvent combinée à des **fonctions de statut** :

| Fonction | Vrai quand… |
|---|---|
| `success()` | Toutes les étapes/jobs précédents ont réussi (comportement **par défaut** implicite si `if:` est absent) |
| `failure()` | Au moins une étape/job précédent a échoué |
| `always()` | Toujours vrai, peu importe le résultat des étapes/jobs précédents (même après un échec ou une annulation) |
| `cancelled()` | Le workflow a été annulé manuellement (ou automatiquement, ex. par un nouveau push qui annule un run en cours) |

### `if:` sur une étape

Cas classique : exécuter une étape de nettoyage ou de notification **même si** une étape
précédente a échoué, ce qu'un `run:` normal ne ferait jamais (le job s'arrête dès la première
étape en échec) :

```yaml
steps:
  - name: Build and run tests
    run: ./mvnw -B clean verify

  - name: Publier le rapport de tests
    if: always()                       # s'exécute même si les tests précédents ont échoué
    uses: actions/upload-artifact@v5
    with:
      name: rapport-surefire
      path: target/surefire-reports/

  - name: Notifier un échec
    if: failure()                      # s'exécute seulement si une étape précédente a échoué
    run: echo "Le build a échoué, vérifier les journaux."
```

> 💡 Sans `if: always()`, l'étape `Publier le rapport de tests` serait **sautée** si `mvnw verify`
> échoue — alors que c'est justement dans ce cas-là que le rapport de tests est le plus utile pour
> diagnostiquer le problème !

### `if:` sur un job

La même logique s'applique à un job entier, combinée à `needs:` pour réagir au résultat d'un job
précédent :

```yaml
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps: [ ... ]

  notify-on-failure:
    needs: build-and-test
    if: failure()                     # ne s'exécute que si build-and-test a échoué
    runs-on: ubuntu-latest
    steps:
      - run: echo "Pipeline en échec, notifier l'équipe."
```

### `if:` avec une condition personnalisée

`if:` accepte aussi des expressions arbitraires (contexte `github.*`, comparaisons), pas seulement
des fonctions de statut :

```yaml
- name: Déployer (seulement sur main)
  if: github.ref_name == 'main' && success()
  run: echo "Déploiement..."
```

<details>
<summary>🤔 Testez-vous</summary>

Une étape `Envoyer une notification Slack en cas d'échec` n'utilise aucune clause `if:`. Le build
échoue à l'étape précédente, mais la notification ne se déclenche jamais. Pourquoi, et comment
corriger ce comportement ?

**Réponse** : sans `if:`, GitHub Actions applique la règle implicite `success()` — l'étape est
**sautée** dès qu'une étape précédente du job a échoué. Il faut ajouter `if: failure()` (ou
`if: always()` si la notification doit aussi apparaître en cas de succès) pour que cette étape
s'exécute précisément quand le build a échoué.
</details>

---

## 📚 Références

- Documentation officielle — [Understanding GitHub Actions](https://docs.github.com/actions/learn-github-actions/understanding-github-actions).
- Documentation officielle — [Workflow syntax for GitHub Actions](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions).
- Documentation officielle — [Expressions (fonctions de statut `success`, `failure`, `always`, `cancelled`)](https://docs.github.com/actions/learn-github-actions/expressions#status-check-functions).
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions) — catalogue d'actions réutilisables.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Intégration continue]({{% relref "/semaine-5/integration-continue" %}}) pour
appliquer ces bases à un pipeline concret qui construit votre projet et exécute sa suite de tests.
{{% /notice %}}
