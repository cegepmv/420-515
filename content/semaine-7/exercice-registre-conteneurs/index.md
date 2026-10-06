+++
title = "Exercice : publier une image sur GitHub Container Registry"
weight = 4
draft = false
+++

## 🎯 Objectif

Automatiser la construction et la publication de votre image Docker dans votre pipeline CI, en
réinvestissant les notions vues dans
[Registres de conteneurs et publication automatisée]({{% relref "/semaine-7/registre-conteneurs" %}}).

---

## 🧩 Partie 1 — Ajouter le job de publication

1. Choisissez une structure (voir [Registres de conteneurs]({{% relref "/semaine-7/registre-conteneurs" %}})) :
   **Structure A** (un seul job autonome qui compile puis construit/publie) ou **Structure B**
   (un job séparé qui dépend du job de tests et récupère son `.jar` via `download-artifact`, sans
   recompiler).
2. Si vous choisissez la Structure B, assurez-vous que ce job dépend du job de tests (`needs:`) —
   l'image ne doit **jamais** être publiée si la suite de tests échoue.
3. Utilisez un `Dockerfile` **runtime seulement** (Option A — voir
   [Docker]({{% relref "/semaine-7/docker" %}})), qui ne fait que copier un `.jar` déjà construit,
   plutôt qu'un `Dockerfile` multi-étapes qui recompilerait une seconde fois inutilement.
4. Configurez les permissions nécessaires (`packages: write`) sur ce job.

---

## 🔐 Partie 2 — Authentification et première publication

1. Ajoutez l'étape `docker/login-action` pour vous authentifier auprès de `ghcr.io`, seulement hors
   Pull Request (`if: github.event_name != 'pull_request'`).
2. Ajoutez l'étape `docker/setup-buildx-action` puis `docker/build-push-action` avec au moins deux
   tags (le SHA du commit et `latest`), en configurant `push: ${{ github.event_name != 'pull_request' }}` —
   l'image se **construit** à chaque Pull Request (pour valider le `Dockerfile`), mais ne se
   **publie** qu'une fois fusionnée sur `main`.
3. Poussez votre branche, ouvrez une Pull Request, et confirmez que l'image se construit sans
   erreur (mais n'apparaît pas encore dans **Packages**). Fusionnez ensuite sur `main` et confirmez
   que l'image apparaît bien dans l'onglet **Packages** de votre dépôt GitHub (ou de votre profil).

---

## 🔍 Partie 3 — Valider l'image publiée

1. Depuis votre machine (ou celle d'un coéquipier), téléchargez l'image publiée :
   `docker pull ghcr.io/<votre-compte>/<votre-depot>:latest`.
2. Démarrez un conteneur à partir de cette image **téléchargée** (pas celle construite localement
   précédemment) et confirmez qu'elle fonctionne de façon identique.
3. Notez le tag correspondant au SHA du commit utilisé, et confirmez que
   `docker pull ghcr.io/<votre-compte>/<votre-depot>:<ce-sha>` fonctionne aussi — cette image ne
   changera plus jamais, même après de futures publications sous `latest`.

<details>
<summary>❓ Question</summary>

Pourquoi est-il important de valider l'image **téléchargée depuis le registre**, plutôt que de se
contenter de l'image construite localement sur votre poste ?

**Réponse** : l'image locale confirme seulement que la construction fonctionne sur **votre**
machine. Télécharger et exécuter l'image depuis le registre valide l'ensemble de la chaîne : que
l'authentification du pipeline fonctionne, que l'image a bien été publiée avec le bon contenu, et
qu'elle s'exécute correctement une fois récupérée ailleurs que sur le poste qui l'a construite —
exactement le scénario qui se produira en production.
</details>

---

## 🏷️ Partie 4 — Visibilité et rétention (discussion d'équipe)

1. Vérifiez la visibilité par défaut de votre image publiée (privée ou publique) dans les
   paramètres du *package* GitHub, et ajustez-la si nécessaire selon les consignes de votre équipe.
2. Discutez en équipe : combien de temps faudrait-il conserver les anciennes images (taguées par
   SHA) avant de les supprimer automatiquement ? Quel risque y a-t-il à les garder indéfiniment
   (espace de stockage) vs. à les supprimer trop vite (perte de traçabilité)?

---

## ✅ Auto-vérification

- [ ] Mon pipeline CI construit et publie automatiquement une image sur GHCR après un merge sur `main`.
- [ ] La publication de l'image dépend explicitement du succès des tests (`needs:`).
- [ ] Mon image est publiée avec au moins deux tags : le SHA du commit et `latest`.
- [ ] J'ai téléchargé et exécuté l'image depuis le registre (pas seulement en local) avec succès.
- [ ] Je sais expliquer la différence entre un tag mobile (`latest`) et un tag immuable (SHA).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[activité pratique]({{% relref "/semaine-7/activite-pratique" %}}) pour
réinvestir l'ensemble de la chaîne (Dockerfile + publication CI/CD) sur votre équipe.
{{% /notice %}}
