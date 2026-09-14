+++
title = "Intégration continue — premier pipeline GitHub Actions / GitLab CI"
weight = 3
draft = true
+++

Écrire de bons tests ne sert à rien s'ils ne sont exécutés que « de temps en temps, quand on y
pense ». L'**intégration continue** (CI) automatise leur exécution à chaque changement de code,
pour détecter une régression **immédiatement**, avant qu'elle n'atteigne la branche principale.

---

## 1️⃣ Pourquoi automatiser

« Ça fonctionne sur ma machine » n'est plus un argument recevable en équipe : la configuration
locale d'une seule personne (version du JDK, variables d'environnement, fichiers non commités) peut
faire passer des tests qui échoueraient ailleurs. Un pipeline CI exécute la suite de tests dans un
environnement **neuf et identique pour tout le monde**, à chaque `push` ou Pull Request — la seule
version de la vérité que toute l'équipe partage.

Un pipeline CI minimal répond à une seule question, de façon automatique et systématique : **le
projet compile-t-il, et tous les tests passent-ils, sur ce commit précis ?**

---

## 2️⃣ Le pipeline « de référence » déjà présent dans le projet

Le dépôt du cours contient déjà, hérités du projet open source `spring-petclinic-rest`, plusieurs
fichiers de workflow GitHub Actions dans `.github/workflows/`. Ils ont été volontairement passés en
déclenchement **manuel** (`workflow_dispatch`) pour ne pas faire échouer votre CI dès le TP1 (ils
référencent des secrets/organisations qui n'existent pas dans le dépôt du cours) — mais ils restent
consultables comme exemple d'un pipeline « niveau industrie » :

```yaml
# .github/workflows/maven-build-pull-request.yml (extrait, déclenchement manuel dans ce dépôt)
name: Java CI for Pull Request (RÉFÉRENCE - déclenchement manuel)

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    - name: Build
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      run: ./mvnw -B verify
```

> 💡 Ce fichier date du projet upstream et déclenche même une analyse SonarCloud
> (`maven-build-master.yml`) — la qualité de code sera abordée en détail à la semaine 9. Le
> pipeline que **vous** construisez cette semaine est plus modeste, mais c'est lui qui protégera
> réellement votre travail d'équipe à partir de maintenant.

---

## 3️⃣ Construire votre propre pipeline — GitHub Actions

Un pipeline minimal qui construit le projet et exécute la suite de tests à chaque Pull Request :

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Build and run tests
        run: ./mvnw -B clean verify
```

| Section | Rôle |
|---|---|
| `on: push` / `on: pull_request` | Événements qui déclenchent le pipeline — ici, à chaque push sur `main` et à chaque Pull Request ciblant `main` |
| `runs-on: ubuntu-latest` | La machine virtuelle (éphémère, neuve à chaque exécution) sur laquelle le job s'exécute |
| `actions/checkout@v4` | Récupère le code du commit concerné sur cette machine virtuelle |
| `actions/setup-java@v4` | Installe le JDK demandé (ici Temurin 17) sur la machine virtuelle |
| `cache: maven` | Met en cache le dépôt local Maven (`~/.m2`) entre les exécutions, pour accélérer les prochains pipelines |
| `./mvnw -B clean verify` | `-B` (*batch mode*, pas d'interaction) ; `verify` déclenche aussi `compile` et `test` (cycle de vie Maven, semaine 3) |

> ⚠️ Utilisez `./mvnw` (le *Maven Wrapper* fourni avec le projet), pas `mvn` directement — la
> machine virtuelle GitHub Actions n'a pas Maven préinstallé dans la version exacte attendue par le
> projet; le wrapper télécharge lui-même la bonne version, garantissant un build reproductible peu
> importe la machine.

### Où voir le résultat

Sur GitHub : onglet **Actions** du dépôt, ou directement sous l'onglet **Checks** d'une Pull
Request. Un ✅ vert ou un ❌ rouge apparaît à côté de chaque commit et de chaque PR.

### Diagnostiquer un échec de pipeline

1. Cliquez sur l'exécution en échec, puis sur le job `build-and-test`.
2. Dépliez l'étape qui a échoué (souvent `Build and run tests`) — la sortie est la **même** console
   qu'un `mvn verify` exécuté localement, avec le rapport Surefire habituel.
3. Reproduisez le problème **localement** avec la même commande (`./mvnw -B clean verify`) avant de
   corriger — évite des allers-retours lents (push → attendre le pipeline → push) pour chaque essai.

<details>
<summary>🤔 Testez-vous</summary>

Votre pipeline échoue sur GitHub Actions, mais `mvn test` passe sans problème sur votre machine.
Quelles sont deux causes plausibles à vérifier en premier ?

**Réponse** : (1) une différence de version du JDK entre votre machine locale et celle configurée
dans `setup-java` du workflow; (2) un fichier nécessaire au build qui n'est pas commité dans le
dépôt (ex. présent seulement en local, ignoré par `.gitignore`) — le pipeline part toujours d'un
clone neuf du dépôt, il ne voit que ce qui est réellement commité.
</details>

---

## 4️⃣ L'équivalent GitLab CI

Si votre équipe utilise GitLab plutôt que GitHub, un pipeline équivalent se définit dans un fichier
`.gitlab-ci.yml` à la racine du dépôt :

```yaml
# .gitlab-ci.yml
image: eclipse-temurin:17-jdk

stages:
  - test

cache:
  paths:
    - .m2/repository

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

build-and-test:
  stage: test
  script:
    - ./mvnw -B clean verify
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

| Section | Rôle |
|---|---|
| `image` | L'image de conteneur Docker (Docker sera vu en détail à la semaine 7) utilisée pour exécuter le job — ici, une image avec le JDK 17 déjà installé |
| `stages` / `build-and-test` | Les grandes étapes du pipeline (ici une seule : `test`) et le job qui s'y exécute |
| `cache` | Équivalent du cache Maven vu côté GitHub Actions |
| `rules` | Conditions de déclenchement — ici, sur chaque Merge Request et sur chaque commit de `main` |

> 💡 Les concepts sont identiques des deux côtés (checkout implicite, environnement neuf,
> installation du JDK, exécution de `mvn verify`, rapport consultable après coup) — seule la
> syntaxe YAML et le vocabulaire (*job*/*stage* vs *step*/*job*) diffèrent.

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Créez le fichier de workflow (GitHub Actions ou GitLab CI, selon la plateforme choisie par votre
équipe), poussez-le sur une branche, et observez l'exécution réelle du pipeline sur votre dépôt.
{{% /notice %}}

---

## 📚 Références

- Documentation officielle GitHub Actions —
  [Building and testing Java with Maven](https://docs.github.com/actions/automating-builds-and-tests/building-and-testing-java-with-maven).
- Documentation officielle GitLab CI/CD —
  [gitlab.com/.../ci/yaml](https://docs.gitlab.com/ee/ci/yaml/).
- Martin Fowler, *Continuous Integration* —
  [martinfowler.com/articles/continuousIntegration.html](https://martinfowler.com/articles/continuousIntegration.html).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Intégration continue]({{% relref "/semaine-5/exercice-integration-continue" %}})
pour mettre votre propre pipeline en place de A à Z.
{{% /notice %}}
