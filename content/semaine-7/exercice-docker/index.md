+++
title = "Exercice : écrire et exécuter un Dockerfile"
weight = 2
draft = false
+++

## 🎯 Objectif

Écrire, construire et exécuter un `Dockerfile` pour votre propre projet, en appliquant les notions
vues dans [Docker — Dockerfile, image, bonnes pratiques]({{% relref "/semaine-7/docker" %}}).

---

## 🧩 Partie 1 — Un premier `Dockerfile` naïf

1. À la racine de votre projet, créez un fichier `Dockerfile` à **une seule étape** (sur le modèle
   de la version naïve vue dans les notes de cette semaine).
2. Construisez l'image : `docker build -t mon-projet:naif .`
3. Notez la taille de l'image obtenue (`docker images` affiche la taille de chaque image).

---

## 🏗️ Partie 2 — Construction multi-étapes (Option B, pour comprendre la technique)

1. Réécrivez votre `Dockerfile` en **construction multi-étapes** : une étape `build` (JDK complet)
   qui produit le `.jar`, puis une étape finale (JRE seulement) qui ne copie que ce `.jar`.
2. Ordonnez vos instructions `COPY` pour exploiter le cache Docker (dépendances avant code source).
3. Reconstruisez l'image sous un nouveau tag : `docker build -t mon-projet:multistage .`
4. Comparez la taille de cette image à celle de la Partie 1.

> 💡 Cet exercice construit et teste localement l'Option B (Dockerfile autosuffisant). Dans
> l'[exercice suivant]({{% relref "/semaine-7/exercice-registre-conteneurs" %}}), votre pipeline CI
> utilisera plutôt l'Option A (réutiliser le `.jar` déjà construit par `build-and-test`), pour
> éviter de compiler le projet deux fois — voir
> [Docker — les deux approches]({{% relref "/semaine-7/docker" %}}) pour la différence.

<details>
<summary>❓ Question</summary>

Vous reconstruisez votre image après avoir modifié un seul fichier `.java` (aucun changement dans
`pom.xml`). Quelles étapes du `Dockerfile` devraient réutiliser le cache Docker, et lesquelles
devraient être réexécutées ?

**Réponse** : si l'ordre des instructions place `COPY pom.xml` et le téléchargement des dépendances
**avant** `COPY src`, alors toutes les instructions jusqu'à (et incluant) le téléchargement des
dépendances réutilisent le cache — seules les instructions à partir de `COPY src` (copie du code
source, compilation) sont réexécutées, puisque c'est le seul fichier qui a changé.
</details>

---

## 🔒 Partie 3 — Utilisateur non-root et `.dockerignore`

1. Ajoutez un utilisateur dédié non-root dans votre `Dockerfile` (`adduser`/`USER`) et confirmez
   que le conteneur démarre toujours correctement.
2. Créez un fichier `.dockerignore` excluant au minimum `target/`, `.git/`, et tout autre dossier
   volumineux ou inutile à la construction.

---

## ▶️ Partie 4 — Exécuter et valider le conteneur

1. Démarrez un conteneur à partir de votre image multi-étapes, en exposant le bon port.
2. Validez le bon fonctionnement avec au moins une requête réelle (`curl` ou Postman) vers un
   endpoint connu de votre application.
3. Arrêtez le conteneur (`docker stop`), puis redémarrez-en un nouveau à partir de la même image —
   confirmez que le comportement est identique (preuve que l'image, pas un état local, définit le
   comportement de l'application).

<details>
<summary>🤔 Testez-vous</summary>

Vous démarrez votre conteneur, mais `curl http://localhost:<port>` échoue avec une erreur de
connexion refusée, alors que l'application affiche bien « Started Application » dans les journaux
du conteneur (`docker logs <nom-du-conteneur>`). Quelle est la cause la plus probable ?

**Réponse** : le port n'a probablement pas été correctement publié au démarrage du conteneur (`-p
<hôte>:<conteneur>` manquant ou mal orthographié), ou le port interne de l'application (`EXPOSE`
dans le `Dockerfile`) ne correspond pas à celui réellement utilisé par l'application (ex.
`server.port` dans `application.properties`). L'application peut tourner **parfaitement** à
l'intérieur du conteneur sans être accessible de l'extérieur si le mappage de port est incorrect.
</details>

---

## ✅ Auto-vérification

- [ ] J'ai un `Dockerfile` en construction multi-étapes, fonctionnel.
- [ ] Mon image finale est basée sur une image JRE (pas JDK complet).
- [ ] Mon `Dockerfile` copie les fichiers de dépendances avant le code source (cache Docker).
- [ ] Le conteneur s'exécute avec un utilisateur non-root.
- [ ] J'ai un fichier `.dockerignore` fonctionnel.
- [ ] J'ai validé mon conteneur avec au moins une requête HTTP réelle.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Registres de conteneurs]({{% relref "/semaine-7/registre-conteneurs" %}}) pour
publier automatiquement cette image depuis votre pipeline CI.
{{% /notice %}}
