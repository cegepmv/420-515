+++
title = "Tags Git et publications (Releases) GitHub"
weight = 2
draft = false
+++

La spécification SemVer ne dit rien sur Git en particulier — mais dans la pratique, le numéro de
version déclaré dans `pom.xml` (ou `package.json`, etc.) est presque toujours **synchronisé** avec
un **tag Git** du même nom (voir [SemVer]({{% relref "/semaine-8/semver" %}})) :

```bash
git tag v4.1.0
git push origin v4.1.0
```

> 💡 Faire correspondre le tag Git et le numéro de version du `pom.xml` (ex. tag `v4.1.0` quand
> `pom.xml` déclare `<version>4.1.0</version>`) permet de retrouver, à partir d'un simple
> `git checkout v4.1.0`, exactement le code source qui correspond à une version publiée — sans
> cette correspondance, un numéro de version devient une simple étiquette non vérifiable.

Voici comment ce tag devient, concrètement, une **Release GitHub** publique — manuellement, puis
automatiquement avec GitHub Actions (voir le pipeline de publication Docker à la
[semaine 7]({{% relref "/semaine-7/registre-conteneurs" %}})).

---

## 1️⃣ La Release GitHub s'appuie directement sur un tag Git

Une **Release GitHub** n'est pas un concept indépendant d'un tag Git : elle s'**appuie
directement** dessus. La documentation officielle le confirme, au sujet de la création manuelle
d'une Release depuis l'interface GitHub :

> « Pour choisir un tag pour la release, utilisez le menu déroulant **Choose a tag**. Pour utiliser
> un tag existant, cliquez sur le tag. Pour créer un nouveau tag, saisissez un numéro de version
> pour votre release, puis cliquez sur **Create new tag**. »
> — [GitHub Docs, *Managing releases in a repository*](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)

Et surtout :

> « Si vous ne sélectionnez pas [*Set as latest release*], l'étiquette de dernière version (*latest
> release*) sera automatiquement attribuée en fonction du versionnage sémantique. »
> — [GitHub Docs, *Managing releases in a repository*](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)

Ce comportement signifie que le tag détermine la Release : GitHub compare les tags `vX.Y.Z`
existants selon les règles SemVer (voir [SemVer, section 2️⃣]({{% relref "/semaine-8/semver" %}}))
pour déterminer automatiquement quel Release afficher comme « *Latest* » — d'où l'importance de
tagger chaque version avec un nom qui respecte strictement ce format.

---

## 2️⃣ Automatiser la création de la Release avec un tag

La séquence complète, du code au fichier publié sur la page **Releases** du dépôt :

```bash
# 1. Mettre à jour le numéro de version (ex. dans pom.xml), puis committer ce changement
git add pom.xml
git commit -m "Bump version to 1.2.3"

# 2. Créer le tag correspondant, PUIS le pousser explicitement
git tag v1.2.3
git push origin v1.2.3
```

Côté workflow, deux éléments (déjà vus à la section « Retour sur GitHub Actions » de
[Docker]({{% relref "/semaine-7/docker" %}})) suffisent à déclencher la création automatique de la
Release **uniquement** quand un tag de ce format est poussé :

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'        # ne se déclenche que sur un tag comme v1.2.3, jamais sur un push ordinaire

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write   # requis pour que l'action puisse créer une Release et y joindre des fichiers

    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Set up JDK 21
        uses: actions/setup-java@v5
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build with Maven
        run: ./mvnw -B package -DskipTests

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v3
        with:
          files: target/*.jar
```

| Élément | Rôle |
|---|---|
| `on: push: tags: 'v*.*.*'` | Déclenche le workflow **seulement** sur un tag correspondant à ce motif |
| `permissions: contents: write` | Sans cette ligne, l'action échoue : le `GITHUB_TOKEN` par défaut n'a pas le droit de créer une Release |
| `softprops/action-gh-release@v3` | Action tierce la plus utilisée pour créer une Release depuis un workflow — reprend **automatiquement** le nom du tag poussé comme nom de la Release, sans configuration additionnelle |
| `with: files: target/*.jar` | Joint le `.jar` compilé comme fichier téléchargeable sur la page de la Release |

> 💡 Par défaut, `softprops/action-gh-release` utilise le tag ayant déclenché le workflow (ex.
> `v1.2.3`) à la fois comme **nom du tag** et comme **titre** de la Release — exactement le
> comportement manuel décrit plus haut, mais automatisé. Le champ `name:` de l'action permet de
> personnaliser ce titre si désiré, sans changer le tag lui-même.

> 💡 Rien n'empêche de combiner ce workflow avec celui de la [semaine 7]({{% relref "/semaine-7/registre-conteneurs" %}})
> qui publie l'image Docker : les deux se déclenchent sur le même tag `vX.Y.Z`, l'un publiant
> l'image sur GHCR, l'autre créant la Release — un même tag peut ainsi piloter plusieurs publications
> en parallèle.

<details>
<summary>🤔 Testez-vous</summary>

Une équipe pousse un tag `release-2024` (sans respecter le format `vX.Y.Z`) sur un dépôt dont le
workflow est configuré avec `on: push: tags: ['v*.*.*']`. Le workflow de Release se déclenche-t-il ?

**Réponse** : non. Le filtre `tags: ['v*.*.*']` n'accepte que les tags qui correspondent **exactement**
à ce motif (ex. `v1.0.0`, `v2.3.1`) — `release-2024` ne commence pas par `v` suivi de trois segments
numériques séparés par des points, donc le workflow ne se déclenche pas du tout.
</details>

---

## 📚 Références

- Documentation officielle GitHub — [Managing releases in a repository](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository).
- Action tierce — [softprops/action-gh-release](https://github.com/softprops/action-gh-release).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Compatibilité d'API : changements compatibles et changements cassants]({{% relref "/semaine-8/compatibilite-api" %}})
pour apprendre à reconnaître, concrètement, ce qui justifie une nouvelle version MAJEURE, MINEURE ou CORRECTIF.
{{% /notice %}}
