+++
title = "Docker — Dockerfile, image, bonnes pratiques"
weight = 1
draft = false
+++

Jusqu'ici, « faire fonctionner » l'application legacy supposait d'avoir installé localement le bon
JDK, Maven, et parfois une base de données compatible. Docker élimine cette dépendance à
l'environnement local : on décrit une seule fois, dans un `Dockerfile`, tout ce qu'il faut pour
exécuter l'application, et cette description produit une **image** identique peu importe la
machine qui l'exécute ensuite.

---

## 0️⃣ Le problème que Docker résout

Imaginez une application composée de plusieurs services (un backend Spring Boot, une base de
données, un frontend React) : chaque service a ses propres exigences de version (JDK, bibliothèque
système, version de Node). Faire cohabiter plusieurs projets aux exigences différentes sur une même
machine, sans qu'ils entrent en conflit, est parfois appelé la **« matrice de l'enfer »** (*Matrix
From Hell*) : chaque nouvelle combinaison de service/version doit être revalidée manuellement, et
rien ne garantit qu'un environnement reproduit fidèlement un autre.

Historiquement, cette incertitude créait une tension classique entre deux rôles :

| Avant Docker | Développeur | Administrateur système |
|---|---|---|
| Responsabilité | Écrit le code, configure son environnement local | Gère les serveurs, configure l'environnement de production |
| Visibilité | Peu de visibilité sur l'infrastructure réelle | Peu de visibilité sur le code |
| Résultat | « Ça fonctionne sur ma machine » | Erreurs découvertes seulement au déploiement |

Docker déplace la définition de l'environnement **dans le code lui-même** (le `Dockerfile`,
versionné comme n'importe quel autre fichier) : la personne qui développe définit l'environnement,
la personne qui déploie exécute **exactement** la même image — l'environnement devient reproductible
plutôt que deviné. C'est en bonne partie ce qui a rendu possible la culture **DevOps** (collaboration
développement/exploitation, automatisation, déploiements fréquents et fiables) à grande échelle.

---

## 1️⃣ Image vs conteneur : le vocabulaire de base

| Terme | Définition |
|---|---|
| **Image** | Un paquet immuable (lecture seule) contenant le code de l'application, son runtime (ex. JRE), ses dépendances et sa configuration — un peu comme une classe en programmation orientée objet |
| **Conteneur** | Une **instance en cours d'exécution** d'une image — comme un objet instancié à partir d'une classe. On peut démarrer plusieurs conteneurs à partir de la même image |
| **`Dockerfile`** | Le fichier texte qui décrit, étape par étape, comment **construire** une image |
| **Registre** (*registry*) | Un serveur qui héberge des images, pour les partager et les télécharger (ex. Docker Hub, GitHub Container Registry — voir la page suivante) |
| **Tag** (étiquette) | Un identifiant de version attaché à une image (ex. `mon-image:1.0`, `mon-image:latest`) |

> 💡 Une image ne change jamais une fois construite. Si vous modifiez le code source, vous devez
> **reconstruire** une nouvelle image (généralement avec un nouveau tag) — vous ne « modifiez »
> jamais une image existante.

---

## 2️⃣ Anatomie d'un `Dockerfile` — version naïve

Un premier jet, qui fonctionne mais contient plusieurs problèmes qu'on va corriger un à un :

```dockerfile
# Dockerfile (naïf — à ne PAS utiliser tel quel)
FROM eclipse-temurin:21
COPY . /app
WORKDIR /app
RUN ./mvnw clean package
CMD ["java", "-jar", "target/spring-petclinic-rest-4.0.2.jar"]
```

| Instruction | Rôle |
|---|---|
| `FROM` | L'image de base sur laquelle construire (ici, une image complète avec le JDK 21) |
| `COPY` | Copie des fichiers de votre machine (contexte de build) vers l'image |
| `WORKDIR` | Définit le répertoire de travail courant pour les instructions suivantes |
| `RUN` | Exécute une commande **pendant la construction** de l'image |
| `CMD` | La commande exécutée par défaut **au démarrage** d'un conteneur basé sur cette image |

Problèmes de cette version :

1. **Image de base trop lourde** : `eclipse-temurin:21` (JDK complet) contient des outils inutiles
   à l'exécution (compilateur, outils de debug) — l'image finale est beaucoup plus grosse que
   nécessaire pour simplement *exécuter* un `.jar` déjà compilé.
2. **Maven est réexécuté dans l'image** : `./mvnw clean package` tourne à l'intérieur du conteneur
   de build, ce qui oblige l'image à contenir Maven, le JDK complet, et tout le cache de
   dépendances — alors que l'exécution finale n'a besoin que du `.jar` et d'un JRE.
3. **Cache Docker mal exploité** : `COPY . /app` copie tout le projet (y compris le code source)
   avant d'exécuter le build — le moindre changement dans n'importe quel fichier invalide le cache
   de **toutes** les étapes suivantes, même si les dépendances Maven n'ont pas changé.

> 💡 Les Option A et B de la section suivante corrigent ce dernier point avec
> `COPY target/*.jar app.jar` : le caractère générique (`*`) tolère n'importe quelle version du
> `.jar`, et le renommer systématiquement en `app.jar` permet d'utiliser `CMD`/`ENTRYPOINT` avec un
> nom **fixe**, qui ne dépend plus jamais du numéro de version du projet.

---

## 3️⃣ Instructions courantes d'un `Dockerfile`

Au-delà de `FROM`, `COPY`, `WORKDIR`, `RUN` et `CMD` déjà vus, voici les instructions les plus
fréquentes dans un `Dockerfile` réel :

| Instruction | Rôle |
|---|---|
| `LABEL` | Ajoute des métadonnées à l'image (version, mainteneur, description) — utile pour l'organisation et l'automatisation, sans effet sur l'exécution |
| `ENV` | Définit une variable d'environnement **permanente**, disponible dans l'image finale et au démarrage du conteneur |
| `ARG` | Définit une variable disponible **seulement pendant la construction** (`docker build --build-arg VERSION=1.2 .`), absente de l'image finale une fois construite |
| `COPY` vs `ADD` | `COPY` copie simplement des fichiers de l'hôte vers l'image; `ADD` fait la même chose mais ajoute des fonctionnalités (extraction automatique d'archives `.tar`, téléchargement depuis une URL) rarement nécessaires — préférez `COPY` sauf besoin explicite |
| `EXPOSE` | **Documente** le port utilisé par l'application à l'intérieur du conteneur — n'ouvre **aucun** port réellement; l'exposition effective se fait uniquement via `-p` au `docker run` |
| `VOLUME` | Déclare un point de montage destiné à des données qui doivent survivre au-delà du cycle de vie du conteneur (ex. fichiers de données, base de données) |

```dockerfile
LABEL version="1.0" maintainer="equipe@exemple.com" description="API TP1 - recherche de personnes"

ENV SPRING_PROFILES_ACTIVE=production

ARG BUILD_VERSION=dev
```

### `RUN` : enchaîner les commandes plutôt que les multiplier

Chaque instruction `RUN` crée une nouvelle **couche** (*layer*) dans l'image. Multiplier les `RUN`
pour des étapes liées gonfle inutilement le nombre de couches :

```dockerfile
# À éviter : trois couches distinctes, dont une qui conserve les fichiers temporaires d'installation
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

```dockerfile
# Préférable : une seule couche, nettoyage inclus dans la même instruction
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

### `CMD` vs `ENTRYPOINT`

Les deux définissent ce qui s'exécute au démarrage d'un conteneur, mais avec une nuance importante :

| | `ENTRYPOINT` | `CMD` |
|---|---|---|
| Rôle | L'exécutable **fixe** du conteneur | Les arguments **par défaut**, remplaçables au `docker run` |
| Remplaçable à l'exécution ? | Non (sauf `--entrypoint`) | Oui — `docker run mon-image autre-commande` remplace le `CMD` |

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--server.port=9966"]     # argument par défaut, modifiable au `docker run`
```

Ici, `ENTRYPOINT` + `CMD` forment ensemble la commande exécutée : `java -jar app.jar --server.port=9966`.
Pour la modifier, on ajoute des arguments après le nom de l'image au `docker run` — ils **remplacent**
entièrement le `CMD` (mais pas l'`ENTRYPOINT`) :

```bash
# Démarre sur le port 8080 au lieu de 9966 (remplace le CMD)
docker run mon-image --server.port=8080
# Commande réellement exécutée : java -jar app.jar --server.port=8080
```

> ⚠️ `EXPOSE 9966` dans un `Dockerfile` ne rend **jamais** l'application accessible à elle seule —
> c'est une documentation à l'intention de la personne qui exécutera le conteneur. Sans
> `-p 9966:9966` au moment du `docker run`, aucune requête externe n'atteint le conteneur, même si
> l'application tourne correctement à l'intérieur.

---

## 4️⃣ Construire l'image à partir du `.jar` : deux approches valables

Il existe deux grandes façons d'organiser un `Dockerfile` autour de la compilation, selon qu'un
**autre** mécanisme (votre pipeline CI) a déjà produit le `.jar` ou non. Choisir la mauvaise
approche peut mener à compiler le projet **deux fois pour rien** — un piège fréquent qui mérite
d'être compris explicitement avant de choisir.

### Option A — Dockerfile « exécution seulement » (recommandée dans ce cours)

Votre pipeline CI (semaine 5) exécute déjà `./mvnw -B clean verify`, ce qui compile le projet,
exécute la suite de tests, **et** produit le `.jar` dans `target/`. Puisque ce `.jar` existe déjà et
a déjà été validé par les tests, le `Dockerfile` n'a **aucune raison** de recompiler quoi que ce
soit — il n'a qu'à l'emballer :

```dockerfile
# Dockerfile — runtime seulement, aucune compilation ici
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY target/*.jar app.jar

EXPOSE 9966
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Ce `Dockerfile` suppose que `target/*.jar` existe déjà **avant** le `docker build` — ce qui est
précisément le cas dans un job de pipeline qui s'exécute après (ou dans la continuité de) votre job
de tests :

```yaml
# Extrait du job build-and-push-image (voir Registres de conteneurs)
steps:
  - uses: actions/checkout@v5
  - name: Récupérer le .jar déjà construit par le job de tests
    uses: actions/download-artifact@v5
    with:
      name: application-jar
      path: target/
  - name: Construire et publier l'image (sans recompiler)
    uses: docker/build-push-action@v6
    with: { context: ., push: true, tags: "..." }
```

> ⚠️ **Un seul `mvn verify` dans tout le pipeline.** Le job `build-and-push-image` **réutilise**
> l'artefact `.jar` publié par `build-and-test` (semaine 5) plutôt que de reconstruire le projet —
> Maven ne s'exécute qu'une seule fois, dans un seul job, pour l'ensemble du pipeline.

### Option B — Construction multi-étapes (Dockerfile autosuffisant)

À l'inverse, si **aucun** autre mécanisme ne produit déjà le `.jar` (ex. un `docker build` exécuté
directement par un développeur en local, sans passer par le pipeline CI, ou un contexte où le
`Dockerfile` doit rester l'unique source de vérité pour construire le projet), le `Dockerfile`
doit alors inclure lui-même l'étape de compilation. C'est le rôle de la **construction
multi-étapes** : une étape de construction (avec le JDK complet et Maven) dont on ne garde que le
résultat (`.jar`), puis une étape finale beaucoup plus légère (JRE seulement) :

```dockerfile
# Dockerfile
# --- Étape 1 : construction ---
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

# Copier d'abord uniquement les fichiers nécessaires au téléchargement des dépendances
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
RUN ./mvnw dependency:go-offline -B

# Copier le code source seulement ensuite, puis construire le .jar
COPY src src
RUN ./mvnw clean package -DskipTests -B

# --- Étape 2 : exécution ---
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 9966
ENTRYPOINT ["java", "-jar", "app.jar"]
```

| Élément | Pourquoi |
|---|---|
| `AS build` | Nomme la première étape pour pouvoir y référer plus tard (`--from=build`) |
| `eclipse-temurin:21-jdk-alpine` (étape 1) | JDK complet nécessaire pour **compiler**, mais basé sur Alpine Linux (très léger) |
| `eclipse-temurin:21-jre-alpine` (étape 2) | Seulement le JRE (nécessaire pour **exécuter** un `.jar` déjà compilé) — image finale beaucoup plus petite |
| `COPY --from=build` | Ne copie **que** le `.jar` produit par l'étape 1 dans l'image finale — le JDK, Maven et le code source de l'étape 1 ne font jamais partie de l'image finale |
| `RUN ./mvnw dependency:go-offline -B` | Télécharge toutes les dépendances Maven déclarées dans `pom.xml` **avant** de copier le code source — l'étape est mise en cache par Docker et n'est réexécutée que si `pom.xml` change |
| `COPY pom.xml` avant `COPY src` | Exploite le cache Docker : tant que `pom.xml` ne change pas, l'étape `dependency:go-offline` reste en cache, même si le code source change constamment |

> 💡 **Ordre des instructions = exploitation du cache.** Docker met en cache chaque instruction
> individuellement; dès qu'une instruction change (ou qu'un fichier qu'elle copie change), **toutes
> les instructions suivantes** perdent leur cache. Placez donc les instructions les plus stables
> (dépendances) **avant** les plus volatiles (code source).

> ⚠️ **Piège de redondance.** Si votre pipeline CI exécute **déjà** `mvn verify` dans un job séparé
> (comme à la semaine 5), et que le job suivant utilise un `Dockerfile` en construction
> multi-étapes, le projet est compilé **deux fois** : une fois par le job de tests, une seconde
> fois à l'intérieur du conteneur de build Docker — un gaspillage de temps de pipeline (et un
> risque, en théorie, que les deux compilations produisent un résultat subtilement différent). Dans
> **ce cours**, préférez l'Option A, qui élimine cette redondance.

### Quand choisir quoi

| Situation | Approche recommandée |
|---|---|
| Le pipeline CI compile déjà le projet dans un job séparé (votre cas, semaine 5 + 7) | **Option A** — Dockerfile runtime seulement, réutilise le `.jar` déjà construit et testé |
| Le `Dockerfile` doit être exécutable de façon autonome, sans dépendre d'un autre job (ex. build local d'un développeur, environnement sans CI) | **Option B** — construction multi-étapes, autosuffisante |

<details>
<summary>🤔 Testez-vous</summary>

Une équipe utilise un pipeline avec un job `build-and-test` (qui exécute `mvn verify`) suivi d'un
job `build-and-push-image` dont le `Dockerfile` est en construction multi-étapes (Option B).
Qu'est-ce qui ne va pas, et comment corriger la situation ?

**Réponse** : le projet est compilé deux fois pour rien — une fois dans `build-and-test`, une
seconde fois dans l'étape de construction du `Dockerfile`. La correction consiste à adopter
l'Option A : le job `build-and-push-image` télécharge le `.jar` déjà produit et validé par
`build-and-test` (via `actions/download-artifact`), et utilise un `Dockerfile` runtime seulement
(sans étape de compilation), qui se contente de copier ce `.jar` existant.
</details>

---

## 5️⃣ Bonnes pratiques supplémentaires


### `.dockerignore`

Tout comme `.gitignore`, un fichier `.dockerignore` à la racine du projet empêche de copier des
fichiers inutiles (ou sensibles) dans le contexte de build :

```
target/
.git/
.idea/
*.log
node_modules/
```

> ⚠️ Sans `.dockerignore`, le dossier `target/` d'un build local précédent pourrait être copié dans
> l'image par erreur (si une instruction `COPY . .` existe quelque part), gonflant inutilement
> l'image ou, pire, y intégrant un `.jar` périmé plutôt que celui fraîchement construit.

### Ne jamais utiliser seulement le tag `latest` en production

`latest` est simplement le nom d'un tag parmi d'autres (pas un tag « spécial » techniquement) —
rien ne garantit qu'il pointe vers une version précise et stable. Préférez un tag explicite (numéro
de version, SHA du commit — voir la page suivante sur les registres) pour toujours savoir **quelle**
version exacte tourne en production.

### Limiter la portée du contexte de build

La commande `docker build .` envoie **tout** le contenu du dossier courant (sauf ce qui est exclu
par `.dockerignore`) au démon Docker avant même de commencer à construire l'image — un dossier
volumineux (ex. avec un gros historique Git ou un `node_modules` non ignoré) ralentit inutilement
chaque build.

<details>
<summary>🤔 Testez-vous</summary>

Pourquoi est-il risqué, du point de vue de la sécurité, de laisser un conteneur s'exécuter avec
l'utilisateur `root` par défaut, même si l'image elle-même ne contient aucun code malveillant ?

**Réponse** : si une vulnérabilité de l'application (ou d'une de ses dépendances) permettait à un
attaquant d'exécuter du code arbitraire à l'intérieur du conteneur, cet utilisateur `root` aurait
tous les privilèges disponibles dans le conteneur (et, selon la configuration de l'hôte, parfois au
delà). Un utilisateur applicatif dédié, sans privilèges superflus, réduit la portée des dégâts
possibles en cas de compromission — un principe de sécurité appelé le *principe du moindre
privilège*.
</details>

---

## 6️⃣ Construire et exécuter l'image localement

```bash
# Construire l'image, avec un tag explicite
docker build -t petclinic-legacy:local .

# Démarrer un conteneur à partir de cette image, en arrière-plan (mode détaché),
# en exposant le port 9966
docker run -d -p 9966:9966 petclinic-legacy:local

# Valider que l'application répond (toujours dans le même terminal, puisque -d
# a rendu la main immédiatement plutôt que d'occuper le terminal)
curl http://localhost:9966/petclinic/api/vets
```

| Option | Rôle |
|---|---|
| `-t petclinic-legacy:local` | Nomme et étiquette l'image construite (`nom:tag`) |
| `-d` | Mode **détaché** (*detached*) : démarre le conteneur en arrière-plan et rend immédiatement la main, plutôt que d'occuper le terminal avec les journaux du conteneur |
| `-p 9966:9966` | Relie le port 9966 de votre machine au port 9966 exposé par le conteneur (`hôte:conteneur`) |

> 💡 Sans `-d`, le terminal reste **occupé** par les journaux du conteneur jusqu'à `Ctrl+C` (qui
> arrête aussi le conteneur) — il faudrait alors un second terminal pour exécuter la commande
> `curl` suivante. Avec `-d`, le conteneur continue à tourner en arrière-plan : utilisez
> `docker stop <conteneur>` (ou `docker ps` pour retrouver son identifiant) pour l'arrêter.

> 💡 Un conteneur arrêté (`Ctrl+C`, ou `docker stop`) ne perd **aucune** donnée de l'image qui l'a
> créé — seul l'état interne du conteneur (ex. des fichiers écrits en cours d'exécution, non
> présents dans l'image) disparaît, sauf si un volume est explicitement monté pour le conserver.

---

## 7️⃣ Commandes Docker utiles au quotidien

Au-delà de `docker build` et `docker run`, quelques commandes reviennent constamment une fois un
conteneur démarré :

| Commande | Rôle |
|---|---|
| `docker ps` | Liste les conteneurs **en cours d'exécution** |
| `docker ps -a` | Liste **tous** les conteneurs, y compris ceux arrêtés |
| `docker logs <conteneur>` | Affiche les journaux (sortie console) d'un conteneur |
| `docker logs -f <conteneur>` | Suit les journaux en temps réel (équivalent de `tail -f`) |
| `docker exec -it <conteneur> /bin/sh` | Ouvre un terminal interactif **à l'intérieur** d'un conteneur déjà démarré |
| `docker stop <conteneur>` | Arrête proprement un conteneur en cours d'exécution |
| `docker rm <conteneur>` | Supprime un conteneur arrêté (`-f` pour forcer la suppression d'un conteneur actif) |
| `docker images` | Liste les images présentes localement, avec leur taille |
| `docker system prune` | Nettoie les ressources Docker inutilisées (conteneurs arrêtés, images non taguées, caches) — utile quand l'espace disque local se remplit |

```bash
# Démarrer en arrière-plan (mode détaché) plutôt que d'occuper le terminal
docker run -d -p 9966:9966 --name petclinic petclinic-legacy:local

# Consulter les journaux de ce conteneur nommé
docker logs -f petclinic

# Ouvrir un terminal à l'intérieur, pour inspecter le système de fichiers de l'image
docker exec -it petclinic /bin/sh
```

> 💡 Nommer explicitement un conteneur (`--name petclinic`) évite d'avoir à retrouver un identifiant
> généré aléatoirement à chaque commande suivante (`docker logs`, `docker stop`, etc.).

---

## 8️⃣ Retour sur GitHub Actions : runners auto-hébergés et variables d'environnement

Avant de construire le pipeline de publication (voir
[Registres de conteneurs]({{% relref "/semaine-7/registre-conteneurs" %}})), il faut revenir sur
deux notions de GitHub Actions vues à la semaine 5 — le **runner** et les **variables** — car
Docker en révèle des subtilités qui ne sautaient pas aux yeux tant qu'on ne faisait que compiler et
tester.

### Runners hébergés par GitHub vs runners auto-hébergés

Le tableau de vocabulaire de la semaine 5 mentionnait déjà le runner comme « la machine virtuelle
sur laquelle un job s'exécute », en précisant qu'elle pouvait être `ubuntu-latest` **ou un runner
auto-hébergé**. Voici la différence concrète entre les deux :

| | Runner **hébergé par GitHub** (`ubuntu-latest`, etc.) | Runner **auto-hébergé** (*self-hosted*) |
|---|---|---|
| Qui le fournit ? | GitHub (infrastructure cloud gérée par GitHub) | Vous (votre propre machine, serveur, VM) |
| Durée de vie | **Éphémère** : détruite et recréée à neuf à chaque exécution de job | **Persistante** : la même machine sert d'un run à l'autre |
| Docker préinstallé ? | ✅ Oui — Docker Engine et Buildx sont déjà présents sur `ubuntu-latest` | ❌ Non — à installer et maintenir vous-même |
| Coût | Inclus (avec des minutes gratuites limitées selon le plan GitHub) | Le vôtre (électricité, matériel, maintenance) |
| Cas d'usage typique | La grande majorité des projets, dont ce cours | Matériel spécifique (GPU, architecture ARM), réseau privé/pare-feu, ou builds très longs qui profitent d'un cache persistant |

> 💡 C'est justement parce qu'`ubuntu-latest` fournit déjà Docker que les exemples de ce chapitre
> (`docker build`, `docker/build-push-action`, etc.) fonctionnent **sans aucune étape
> d'installation préalable** dans le workflow.

> ⚠️ Un runner auto-hébergé n'est **pas** recréé à neuf entre deux exécutions : un fichier laissé
> par un job précédent, une variable d'environnement exportée dans le shell, ou même une ancienne
> image Docker encore en cache peuvent **fuiter** d'un run à l'autre. C'est un gain de rapidité
> (rien à retélécharger), mais au prix de la garantie de reproductibilité stricte qu'offre un
> runner hébergé — une raison de plus, sur un runner auto-hébergé, de ne jamais présumer de l'état
> initial de la machine dans un workflow.

### Créer et enregistrer son propre runner auto-hébergé

GitHub ne fournit pas de machine : c'est au propriétaire du dépôt (ou de l'organisation) d'en
préparer une (physique, VM, conteneur) et de l'**enregistrer** auprès de GitHub. Les grandes étapes
sont toujours les mêmes, peu importe l'OS :

1. Dans le dépôt (ou l'organisation), aller dans **Settings → Actions → Runners → New self-hosted
   runner**.
2. Choisir le système d'exploitation et l'architecture de la machine qui servira de runner (Linux,
   Windows, macOS — x64 ou ARM64).
3. GitHub génère alors une série de commandes **propres à ce dépôt** (incluant un jeton
   d'enregistrement temporaire) à exécuter sur la machine cible, par exemple sur Linux :

   ```bash
   # 1. Télécharger le paquet du runner
   mkdir actions-runner && cd actions-runner
   curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/download/v2.XXX.X/actions-runner-linux-x64-2.XXX.X.tar.gz
   tar xzf ./actions-runner.tar.gz

   # 2. Enregistrer ce runner auprès du dépôt (URL + jeton fournis par GitHub)
   ./config.sh --url https://github.com/<compte>/<depot> --token <JETON_TEMPORAIRE>

   # 3. Démarrer le runner (reste attaché au terminal — voir l'option service ci-dessous)
   ./run.sh
   ```

4. Pendant `./config.sh`, GitHub propose d'associer une ou plusieurs **étiquettes** (*labels*) au
   runner (ex. `self-hosted`, `linux`, `gpu`) — ce sont ces étiquettes qui permettent ensuite de le
   cibler précisément dans un workflow :

   ```yaml
   jobs:
     build-and-push-image:
       runs-on: self-hosted        # ou [self-hosted, linux, gpu] pour cibler des étiquettes précises
   ```

5. Pour que le runner reste actif en permanence (plutôt que seulement tant que `./run.sh` occupe un
   terminal ouvert), l'installer comme **service système** :

   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

6. Le runner apparaît ensuite comme **Idle** (en attente) dans **Settings → Actions → Runners** —
   signe qu'il est prêt à recevoir des jobs ciblant son étiquette.

> ⚠️ **Ne jamais** configurer un runner auto-hébergé sur un **dépôt public** sans restriction
> supplémentaire : n'importe qui peut ouvrir une Pull Request depuis un fork, et son contenu (y
> compris du code arbitraire dans un `run:`) s'exécuterait alors directement sur **votre** machine.
> GitHub recommande de limiter ce risque via les paramètres d'approbation des workflows provenant
> de forks, ou de réserver les runners auto-hébergés aux dépôts privés/internes.

### Variables d'environnement : trois façons de les définir

La section 5️⃣ de la semaine 5 montrait déjà les expressions `${{ }}` et les secrets. Les
**variables d'environnement** (`env:`) sont un troisième mécanisme, complémentaire : des valeurs
simples (non secrètes) injectées dans l'environnement shell d'une étape, accessibles comme
n'importe quelle variable d'environnement système.

`env:` peut être défini à trois niveaux, du plus large au plus précis :

```yaml
env:                                    # (1) niveau workflow : visible par tous les jobs
  IMAGE_NAME: petclinic

jobs:
  build-and-push-image:
    env:                                # (2) niveau job : visible par toutes les étapes de ce job
      REGISTRY: ghcr.io
    runs-on: ubuntu-latest
    steps:
      - name: Afficher les variables
        env:                            # (3) niveau étape : visible seulement par cette étape
          TAG: latest
        run: echo "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.TAG }}"
```

| Niveau | Portée |
|---|---|
| `env:` au niveau du **workflow** | Toutes les étapes de tous les jobs du fichier |
| `env:` au niveau du **job** | Toutes les étapes de ce job seulement |
| `env:` au niveau de la **step** | Cette étape seulement |

> 💡 Une variable définie à un niveau plus précis (étape) **masque** une variable de même nom
> définie à un niveau plus large (job ou workflow) — même logique de portée qu'une variable locale
> qui cache une variable globale dans la plupart des langages de programmation.

### `$GITHUB_ENV` : faire passer une valeur d'une étape à la suivante

Chaque étape d'un job s'exécute dans un processus shell **distinct** : une variable exportée avec
`export` dans une étape (`run:`) **ne survit pas** jusqu'à l'étape suivante. Pour faire passer une
valeur calculée dynamiquement (ex. le tag Docker à utiliser), GitHub Actions fournit le fichier
spécial `$GITHUB_ENV` :

```yaml
steps:
  - name: Calculer le tag de l'image
    run: echo "IMAGE_TAG=$(date +%Y%m%d)-${{ github.sha }}" >> "$GITHUB_ENV"

  - name: Utiliser le tag calculé
    run: echo "Construction de l'image avec le tag ${{ env.IMAGE_TAG }}"
```

Tout ce qui est **ajouté** (`>>`, jamais `>` qui écraserait le fichier) à `$GITHUB_ENV` au format
`CLÉ=valeur` devient disponible comme variable d'environnement (`env.CLÉ`) dans **toutes les
étapes suivantes** du même job — mais pas dans les autres jobs, chacun démarrant avec un
environnement vide (ou hérité uniquement du `env:` déclaré au niveau du workflow).

### Variables (`vars`) vs secrets (`secrets`) : non-secret ne veut pas dire « à coder en dur »

GitHub distingue aussi les **variables de dépôt** (*Settings → Secrets and variables → Actions →
Variables*), accessibles via le contexte `vars`, des secrets déjà vus à la semaine 5 :

| | `vars.*` (variables) | `secrets.*` (secrets) |
|---|---|---|
| Visible en clair dans les journaux ? | Oui | Non — automatiquement masqué (`***`) |
| Cas d'usage | Nom d'image, région de déploiement, URL d'un environnement — une valeur de configuration, pas sensible | Jeton d'API, mot de passe, clé privée |
| Où le configurer | *Settings → Secrets and variables → Actions → Variables* | *Settings → Secrets and variables → Actions → Secrets* |

```yaml
- name: Construire l'image
  run: docker build -t ${{ vars.REGISTRY }}/${{ github.repository }}:latest .
```

> ⚠️ L'intérêt de `vars.*` par rapport à une valeur codée en dur directement dans le YAML n'est pas
> la confidentialité (ce n'est pas un secret) mais la **centralisation** : changer la valeur une
> seule fois dans les paramètres du dépôt, plutôt que de modifier chaque fichier de workflow qui
> l'utilise.

<details>
<summary>🤔 Testez-vous</summary>

Votre équipe migre son pipeline d'un runner `ubuntu-latest` vers un runner auto-hébergé, pour
profiter d'un cache Docker persistant entre les exécutions et accélérer les builds. Un coéquipier
remarque qu'une variable `REGISTRY=ghcr.io` définie dans un ancien job de test, il y a plusieurs
semaines, apparaît encore dans les journaux d'un job complètement différent qui ne la définit nulle
part. Comment est-ce possible, et ce risque existait-il avec `ubuntu-latest` ?

**Réponse** : sur un runner **auto-hébergé**, la machine n'est **jamais recréée à neuf** entre deux
exécutions — un export laissé dans le shell d'un run précédent (ex. via un script appelé par
`run:`, en dehors du mécanisme `env:`/`$GITHUB_ENV` propre à GitHub Actions) peut persister dans
l'environnement du processus et réapparaître dans un run ultérieur. Sur un runner **hébergé par
GitHub** (`ubuntu-latest`), ce risque n'existe pas : chaque job démarre sur une machine virtuelle
entièrement neuve, détruite après son exécution — aucun état, variable ou fichier ne peut
« fuiter » d'un run à l'autre.
</details>

### `permissions:` et déclencheurs par tag : vers les Releases GitHub

Deux derniers concepts GitHub Actions, utiles au-delà de Docker, complètent ce retour en arrière :

- **`permissions:`** — par défaut, le jeton `secrets.GITHUB_TOKEN` fourni automatiquement à chaque
  run n'a que des droits limités en lecture. Certaines actions (ex. créer une Release, commenter
  une Pull Request) exigent un droit d'écriture explicite, déclaré au niveau du job :

  ```yaml
  jobs:
    release:
      permissions:
        contents: write     # requis, par exemple, pour que le workflow puisse créer une Release
  ```

- **Déclencher sur un tag plutôt qu'un `push` ordinaire** — en plus de `branches:` (section 3️⃣),
  `on: push:` accepte un filtre `tags:` : le workflow ne se déclenche alors **que** lorsqu'un tag
  correspondant (ex. `v1.2.3`) est poussé, jamais sur un commit ordinaire vers `main` :

  ```yaml
  on:
    push:
      tags:
        - 'v*'
  ```

  Ce tag ne se crée **jamais tout seul** : il faut le créer puis le pousser explicitement depuis
  votre poste, séparément d'un `git push` ordinaire de commits — c'est cette action volontaire, et
  seulement elle, qui déclenche le workflow :

  ```bash
  git tag v1.0.0
  git push origin v1.0.0          # déclenche le workflow — un simple "git push origin main" ne le ferait pas
  ```

> 💡 Ces deux éléments (`permissions: contents: write` + déclencheur `tags:`) sont précisément ce
> qui permet d'automatiser la création d'une **Release GitHub** (page de version avec notes et
> fichiers joints, ex. le `.jar` compilé) à chaque tag poussé — voir l'exemple complet dans
> [Registres de conteneurs et publication automatisée]({{% relref "/semaine-7/registre-conteneurs" %}}),
> section « Créer une publication GitHub (*Release*) automatiquement ».

### Badge de statut : afficher l'état du pipeline dans le `README`

GitHub génère automatiquement, pour chaque workflow, un **badge** SVG affichant son dernier statut
(✅ réussi, ❌ échoué) — habituellement affiché tout en haut du `README.md` d'un projet, pour que
quiconque visite le dépôt voie immédiatement si la CI passe, sans avoir à ouvrir l'onglet
**Actions** :

```markdown
![CI](https://github.com/<compte>/<depot>/actions/workflows/ci.yml/badge.svg)
```

| Partie de l'URL | Rôle |
|---|---|
| `<compte>/<depot>` | Le dépôt GitHub concerné |
| `workflows/ci.yml` | Le **nom du fichier** de workflow (pas son `name:` affiché dans l'interface) dont on veut montrer le statut |
| `badge.svg` | Demande l'image SVG plutôt que la page HTML de l'exécution |

On peut aussi cibler une branche précise (utile si `main` et une branche de développement ont des
statuts différents) :

```markdown
![CI](https://github.com/<compte>/<depot>/actions/workflows/ci.yml/badge.svg?branch=main)
```

> 💡 Ce badge n'est **pas une image statique** : GitHub la régénère à la volée à chaque fois qu'elle
> est chargée, en fonction du résultat de la **dernière exécution** du workflow ciblé — aucune
> configuration supplémentaire n'est nécessaire au-delà de coller ce lien Markdown dans le
> `README.md`.

> ⚠️ Sur un dépôt **privé**, ce badge n'est visible que par les personnes ayant déjà accès au
> dépôt (elles doivent être authentifiées sur GitHub dans leur navigateur) — aucune **permission**
> supplémentaire à configurer dans le workflow lui-même, contrairement à `contents: write` ou
> `packages: write` vus plus haut : l'accès au badge suit simplement les permissions **du dépôt**,
> pas celles du `GITHUB_TOKEN`. Sur un dépôt **public**, le badge est visible par n'importe qui,
> même sans compte GitHub.

<details>
<summary>🤔 Testez-vous</summary>

Le badge de statut dans le `README` de votre projet affiche ❌ (échec), mais vous venez de corriger
le bogue et de pousser le correctif il y a 10 secondes à peine. Devriez-vous vous inquiéter ?

**Réponse** : pas nécessairement, pas encore. Le badge reflète le résultat de la **dernière
exécution terminée** du workflow — si votre correctif vient tout juste d'être poussé, le nouveau
run est probablement encore **en cours** (le badge affichera temporairement l'ancien statut jusqu'à
ce que ce nouveau run se termine). Il faut attendre la fin de cette nouvelle exécution avant de
considérer le badge comme à jour.
</details>

---

## 📚 Références

- Documentation officielle Docker — [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/).
- Documentation officielle Docker — [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/).
- Documentation officielle Docker — [Dockerfile reference](https://docs.docker.com/reference/dockerfile/).
- Images officielles Eclipse Temurin — [hub.docker.com/_/eclipse-temurin](https://hub.docker.com/_/eclipse-temurin).
- Documentation officielle — [About self-hosted runners](https://docs.github.com/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners).
- Documentation officielle — [Variables (vars, env, $GITHUB_ENV)](https://docs.github.com/actions/learn-github-actions/variables).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Docker]({{% relref "/semaine-7/exercice-docker" %}}) pour écrire,
construire et exécuter votre propre `Dockerfile` sur votre projet.
{{% /notice %}}
