+++
title = "Défi Git — semaine 1"
weight = 8
+++

## 🎯 Objectif

Vous avez déjà utilisé Git par le passé. Plutôt que de refaire un dépôt d'entraînement fictif,
vous allez directement mettre les mains dans le **vrai projet legacy** du cours en équipe
(configuration, premier clone, création de branche, commits, push/pull). Ce défi sert à vérifier
et consolider vos réflexes Git 1/2 (sans Pull Request — ça, c'est pour la semaine 2) pendant que
vous travaillez sur le projet réel.

## 🧩 Le défi

En équipe, sur le dépôt du projet legacy qui vous a été remis :

1. Configurez Git localement (si ce n'est pas déjà fait) et clonez le dépôt du projet.
2. Créez une branche à votre nom ou à celui de votre tâche (ex. `exploration-<votre-nom>`).
3. Faites au moins **un commit** sur cette branche (par exemple, une note d'exploration sur le
   code, un TODO, ou une petite correction repérée en lisant le projet).
4. Poussez votre branche vers le dépôt distant.
5. Vérifiez avec `git pull` que votre copie locale de `main` reste synchronisée avec le travail de
   vos coéquipiers.

**Ne faites pas de pull request pour l'instant** — vous verrez comment (et pourquoi) l'utiliser à
la semaine 2, en équipe, avec un vrai coéquipier qui révise votre travail.

## ❓ Questions défi

Répondez à ces questions en équipe avant/pendant que vous manipulez le dépôt. Cliquez pour révéler
la réponse une fois que vous avez essayé.

<details>
<summary><strong>1.</strong> Quelle commande configure votre identité Git (nom + courriel) une seule fois pour toutes vos futures utilisations sur ce poste ?</summary>

```bash
git config --global user.name "Votre Nom"
git config --global user.email "vous@example.com"
```

Le drapeau `--global` évite de devoir répéter la configuration à chaque nouveau dépôt.
</details>

<details>
<summary><strong>2.</strong> Vous venez de cloner le projet legacy. Comment créez-vous une nouvelle branche et vous y déplacez en une seule commande ?</summary>

```bash
git checkout -b exploration-<votre-nom>
# équivalent plus récent :
git switch -c exploration-<votre-nom>
```
</details>

<details>
<summary><strong>3.</strong> Vous avez modifié un fichier. Quelles sont les étapes pour l'enregistrer dans un commit ?</summary>

```bash
git add <fichier>       # ou git add . pour tout ajouter
git commit -m "message clair et descriptif"
```

Un bon message de commit décrit **pourquoi**, pas seulement **quoi**.
</details>

<details>
<summary><strong>4.</strong> Comment envoyez-vous votre branche vers le dépôt distant pour la première fois ?</summary>

```bash
git push -u origin exploration-<votre-nom>
```

Le `-u` (upstream) lie votre branche locale à la branche distante, pour que les prochains `git
push`/`git pull` fonctionnent sans préciser `origin`/le nom de branche.
</details>

<details>
<summary><strong>5.</strong> Un coéquipier a poussé des changements sur <code>main</code> pendant que vous travailliez sur votre branche. Comment récupérez-vous ces changements ?</summary>

```bash
git checkout main
git pull
```

Ensuite, si vous voulez intégrer ces changements dans votre propre branche, vous pouvez faire un
`git merge main` (ou `git rebase main`) depuis votre branche.
</details>

<details>
<summary><strong>6.</strong> Quelle commande vous montre rapidement l'état de vos fichiers (modifiés, ajoutés, non suivis) avant de faire un commit ?</summary>

```bash
git status
```

À utiliser par réflexe avant chaque `add`/`commit` pour éviter les surprises.
</details>

## ✅ Auto-vérification en équipe

- [ ] Chaque membre a configuré son identité Git localement.
- [ ] Le dépôt du projet legacy est cloné chez tous les membres de l'équipe.
- [ ] Au moins une branche autre que `main` existe sur le dépôt distant.
- [ ] Au moins un commit a été poussé sur cette branche.
- [ ] `git pull` sur `main` fonctionne sans erreur pour tous les membres.

Voir aussi la page [Introduction à Git (1/2)]({{< relref "/semaine-1/git" >}}) pour les
explications détaillées de chaque commande.
