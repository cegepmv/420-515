+++
title = "Défi Git — semaine 2 (avancé)"
weight = 7
+++

## 🎯 Objectif

Avant de vous lancer dans le **TP1**, vous allez
pratiquer, sans enjeu de note, les réflexes de [Git 2/2]({{< relref "/semaine-2/git-avance" >}}) —
`merge` vs `rebase`, Pull Request, `revert` vs `reset`, `cherry-pick` et `stash` — directement dans
le **vrai projet legacy** (`spring-petclinic-rest-legacy-a26`) que vous explorez depuis la semaine
1. Pour éviter tout risque d'interférer avec le vrai code source (que vous devrez corriger plus
tard dans la session), toutes les manipulations ci-dessous se font dans un seul fichier
d'entraînement à la racine du projet : `PRATIQUE-GIT.md`.

> 👥 **En équipe de 2.** Si vous êtes seul·e, simulez les deux rôles (« Personne A » / « Personne
> B ») avec deux clones locaux du même dépôt (deux dossiers), ou en changeant temporairement
> l'identité Git entre vos commits :
> ```bash
> git commit -m "message" --author="Personne B <personneb@example.com>"
> ```

---

## 🧩 Partie 1 — Provoquer et résoudre un vrai conflit de merge

1. À partir de `main`, créez le fichier d'entraînement (une seule personne le fait, puis le pousse) :
   ```bash
   git checkout main
   git pull
   echo "# Pratique Git — semaine 2" > PRATIQUE-GIT.md
   echo "Ligne de base." >> PRATIQUE-GIT.md
   git add PRATIQUE-GIT.md
   git commit -m "Ajout du fichier de pratique Git"
   git push
   ```
2. **Personne A** crée une branche et modifie la même ligne :
   ```bash
   git checkout -b conflit-a
   ```
   Remplacez la ligne 2 par `Modification de la personne A.`, puis committez et poussez.
3. **Personne B** crée sa propre branche **à partir de `main`** (pas de `conflit-a`) et modifie la
   **même ligne** différemment :
   ```bash
   git checkout main
   git checkout -b conflit-b
   ```
   Remplacez la ligne 2 par `Modification de la personne B.`, puis committez et poussez.
4. Fusionnez `conflit-a` dans `main` (via une Pull Request — voir Partie 2), puis tentez de fusionner
   `conflit-b` :
   ```bash
   git checkout main
   git pull
   git merge conflit-b
   ```
   Vous devriez obtenir un **conflit**. Résolvez-le manuellement (choisissez une version, combinez
   les deux, ou utilisez `git checkout --ours`/`--theirs`), puis terminez :
   ```bash
   git add PRATIQUE-GIT.md
   git commit
   ```

<details>
<summary>❓ Question — pourquoi le conflit apparaît-il ici et pas ailleurs dans le projet ?</summary>

Parce que **les deux branches modifient exactement la même ligne** du même fichier depuis le même
point de départ. Git sait fusionner automatiquement des changements sur des lignes différentes,
mais pas deux changements incompatibles sur la même ligne — il vous laisse trancher.
</details>

---

## 🔀 Partie 2 — Pull Request et revue de code croisée

1. Ouvrez une Pull Request sur GitHub pour `conflit-a` (ou `conflit-b`) vers `main`, avec une
   description claire de ce qui a été changé et pourquoi.
2. Demandez à votre coéquipier·ère (ou à une autre équipe, si vous êtes seul·e) de **réviser** la
   PR : au moins un commentaire, puis l'approbation avant la fusion.
3. Fusionnez la PR depuis l'interface GitHub.

---

## 🪃 Partie 3 — Même scénario, mais avec `rebase`

Reproduisez un scénario similaire à la Partie 1, mais cette fois en gardant un historique linéaire :

```bash
git checkout main
git pull
git checkout -b rebase-pratique
# faites 2-3 commits sur PRATIQUE-GIT.md
git checkout main
git pull            # si main a bougé entretemps
git checkout rebase-pratique
git rebase main
```

Comparez `git log --oneline --graph --all` **avant** et **après** le rebase. Si un conflit survient
pendant le rebase, résolvez-le fichier par fichier, puis :
```bash
git add PRATIQUE-GIT.md
git rebase --continue
```

> ⚠️ Ne poussez jamais un `git push --force` sur une branche que quelqu'un d'autre a déjà récupérée
> — ici, `rebase-pratique` reste une branche personnelle, donc c'est sécuritaire.

---

## ⏪ Partie 4 — `revert` vs `reset`

1. Sur votre branche, faites un commit **volontairement mauvais** (ex. une faute qui casse une
   phrase dans `PRATIQUE-GIT.md`), poussez-le.
2. Corrigez-le de façon **sûre**, sans réécrire l'historique déjà partagé :
   ```bash
   git revert HEAD
   git push
   ```
3. Faites un **deuxième** commit mauvais, mais **ne le poussez pas** cette fois. Annulez-le
   localement avec :
   ```bash
   git reset --soft HEAD~1     # garde les changements en attente
   # ou
   git reset --hard HEAD~1     # supprime aussi les changements — à utiliser avec prudence !
   ```

<details>
<summary>❓ Question — pourquoi utiliser <code>revert</code> plutôt que <code>reset --hard</code> une fois le commit poussé ?</summary>

Parce que `reset --hard` **réécrit l'historique** : si quelqu'un d'autre a déjà récupéré (`pull`)
ce commit, son historique local ne correspondra plus au vôtre après un `push --force`, ce qui
casse la collaboration. `revert` ajoute un nouveau commit qui annule les changements, sans jamais
réécrire ce qui a déjà été partagé.
</details>

---

## 🍒 Partie 5 — Cherry-pick

1. **Personne A** crée une branche avec **trois commits** sur `PRATIQUE-GIT.md` : un commit utile
   (ex. une section « Glossaire Git » à ajouter), et deux commits « bruit » (ex. des notes
   personnelles de brouillon sans intérêt pour l'équipe).
   ```bash
   git checkout -b brouillon-a
   # commit 1 : section utile
   # commit 2 : bruit
   # commit 3 : bruit
   git push -u origin brouillon-a
   ```
2. **Personne B** récupère uniquement le commit utile, **sans fusionner toute la branche** :
   ```bash
   git checkout main
   git pull
   git log --oneline brouillon-a      # repérer le hash du commit utile
   git cherry-pick <hash-du-commit-utile>
   git push
   ```

---

## 📦 Partie 6 (bonus) — `git stash`

Simulez une interruption : vous modifiez `PRATIQUE-GIT.md` sans avoir encore committé, et on vous
demande de changer de branche en urgence pour regarder autre chose.

```bash
git stash                 # met de côté vos changements non commités
git checkout main         # ou une autre branche, sans perdre votre travail en cours
# ... votre urgence ...
git checkout rebase-pratique
git stash pop             # récupère vos changements mis de côté
```

---

## ✅ Auto-vérification en équipe

- [ ] Un vrai conflit de merge a été provoqué **et** résolu manuellement sur `PRATIQUE-GIT.md`.
- [ ] Au moins une Pull Request a été ouverte, révisée par une autre personne, puis fusionnée.
- [ ] Le scénario a été refait avec `rebase`, et l'historique linéaire a été observé avec
      `git log --oneline --graph --all`.
- [ ] `git revert` a été utilisé sur un commit déjà poussé, et `git reset` sur un commit encore
      local seulement.
- [ ] `git cherry-pick` a permis de récupérer un seul commit utile sans fusionner toute une branche.
- [ ] (Bonus) `git stash` / `git stash pop` ont été testés.

> 💡 Gardez ces branches d'entraînement (`conflit-a`, `conflit-b`, `rebase-pratique`,
> `brouillon-a`) — elles pourront être supprimées une fois l'exercice validé, sans affecter le vrai
> code du projet.
