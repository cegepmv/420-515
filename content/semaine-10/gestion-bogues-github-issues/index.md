+++
title = "Gestion des bogues avec GitHub Issues"
weight = 2
draft = false
+++

Tout ce qui suit est fondé sur la documentation officielle de **GitHub Docs** : les pages
« About issues », « Linking a pull request to an issue » et « Managing labels / About milestones ».

> 💡 Ce cours utilise exclusivement **GitHub** (Actions, registre de conteneurs, et maintenant
> suivi de bogues). GitLab propose un outil équivalent (*GitLab Issues*), mais il n'est pas couvert
> ici, de la même façon que GitLab CI n'a pas été couvert à la semaine 5.

---

## 1️⃣ Pourquoi un outil dédié plutôt qu'un fichier `TODO.md`

Jusqu'ici dans ce cours, un bogue découvert sur l'application legacy était généralement traité
immédiatement : reproduit, isolé (semaine 2-3), corrigé, testé (semaine 4-5). En pratique, une
équipe ne peut pas toujours corriger un bogue au moment où il est signalé : la personne qui le
découvre n'est pas forcément celle qui le corrigera, plusieurs bogues doivent être priorisés les
uns par rapport aux autres, et un client ou un utilisateur doit souvent être tenu informé de l'état
d'avancement. Un simple commentaire `// TODO: corriger ce bug` dans le code, ou un message
Slack, se perd rapidement et n'est visible par personne d'autre que celui qui l'a écrit.

GitHub Issues répond à ce besoin par un outil de suivi persistant, intégré directement au dépôt :

> « Vous pouvez créer des issues dans votre dépôt pour planifier, discuter et suivre le travail.
> Les issues sont rapides à créer, flexibles, et peuvent être utilisées de nombreuses façons. Les
> issues peuvent suivre des rapports de bogues, des demandes de nouvelles fonctionnalités et des
> idées, et tout ce que vous devez noter ou discuter avec votre équipe. »
> — [GitHub Docs — About issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/learning-about-issues/about-issues)

Une *issue* GitHub est, au minimum, un titre, une description en Markdown, et un fil de discussion
(commentaires) — mais sa valeur vient surtout des métadonnées qui s'y rattachent, détaillées dans
les sections suivantes : étiquettes, jalons, et liens vers les Pull Requests qui la referment.

---

## 2️⃣ Étiquettes (*labels*) : classer sans ambiguïté

Une **étiquette** catégorise une issue de façon visible et filtrable. GitHub fournit un ensemble
d'étiquettes par défaut dans chaque nouveau dépôt :

> « GitHub fournit des étiquettes par défaut dans chaque nouveau dépôt. Vous pouvez utiliser ces
> étiquettes par défaut pour aider à créer un flux de travail standard dans un dépôt. »
> — [GitHub Docs — Managing labels](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels)

| Étiquette par défaut | Signification officielle |
|---|---|
| `bug` | Indique un problème inattendu ou un comportement non voulu |
| `enhancement` | Indique une demande de nouvelle fonctionnalité |
| `documentation` | Indique un besoin d'amélioration ou d'ajout à la documentation |
| `duplicate` | Indique des issues, Pull Requests ou discussions similaires |
| `good first issue` | Indique une bonne issue pour les personnes qui contribuent pour la première fois |
| `help wanted` | Indique qu'une personne responsable du projet souhaite de l'aide sur une issue ou une Pull Request |
| `invalid` | Indique qu'une issue n'est plus pertinente |
| `question` | Indique qu'une issue a besoin de plus d'informations |
| `wontfix` | Indique que le travail ne se poursuivra pas sur une issue |

Rien n'empêche une équipe de créer ses propres étiquettes en plus de celles-ci — par exemple,
`priorité-haute`, `semaine-9` (pour relier une issue à son origine dans le cours), ou
`régression` pour distinguer un bogue qui fonctionnait avant d'un défaut qui a toujours existé.

---

## 3️⃣ Jalons (*milestones*) : regrouper par échéance

Un **jalon** regroupe plusieurs issues et Pull Requests autour d'un objectif commun, typiquement
une date ou une version :

> « Vous pouvez utiliser les jalons pour suivre la progression de groupes d'issues ou de Pull
> Requests dans un dépôt. [...] Depuis la page du jalon, vous pouvez voir : une description fournie
> par l'utilisateur [...]; la date d'échéance du jalon; le pourcentage d'achèvement du jalon; le
> nombre d'issues et de Pull Requests ouvertes et fermées associées au jalon. »
> — [GitHub Docs — About milestones](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/about-milestones)

**Exemple concret, transposé au cours** : une équipe crée un jalon `TP3 — Qualité et sécurité`
(semaine 9), y associe toutes les issues correspondant aux *issues* SonarCloud et aux
vulnérabilités Trivy à corriger, et voit directement, sur la page du jalon, combien il en reste à
fermer avant la date de remise — sans avoir à relire chaque issue individuellement.

---

## 4️⃣ Lier une Pull Request à une issue

Le mécanisme le plus utile au quotidien reste le **lien automatique** entre une Pull Request et
l'issue qu'elle corrige, par un mot-clé dans la description de la Pull Request (ou dans un message
de commit) :

> « Vous pouvez lier une Pull Request à une issue en utilisant un mot-clé pris en charge dans la
> description de la Pull Request [...]. Lorsque vous fusionnez une Pull Request liée dans la
> **branche par défaut** d'un dépôt, son issue liée est automatiquement fermée. »
> — [GitHub Docs — Linking a pull request to an issue](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue)

| Mots-clés reconnus | Syntaxe (même dépôt) | Syntaxe (dépôt différent) |
|---|---|---|
| `close`, `closes`, `closed` | `Closes #10` | `Closes owner/repo#10` |
| `fix`, `fixes`, `fixed` | `Fixes #10` | `Fixes owner/repo#10` |
| `resolve`, `resolves`, `resolved` | `Resolves #10` | `Resolves owner/repo#10` |

> ⚠️ La documentation précise une limite importante : « Les mots-clés spéciaux dans une description
> de Pull Request ne sont interprétés que lorsque la Pull Request cible la branche **par défaut**
> du dépôt. Si la Pull Request cible une **autre branche**, ces mots-clés sont ignorés, aucun lien
> n'est créé, et fusionner la PR n'a aucun effet sur les issues. » Une Pull Request vers une branche
> `release` (voir le rappel sur les tags à la semaine 8) ne fermerait donc **pas** automatiquement
> l'issue visée par son `Closes #10` — seule une fusion vers la branche par défaut (`main`) le
> ferait.

**Exemple concret** : une Pull Request corrigeant l'injection SQL identifiée par SonarCloud
(semaine 9) pourrait avoir pour description :

```markdown
Corrige la requête construite par concaténation dans `OwnerController`, en la remplaçant par une
requête paramétrée (`PreparedStatement`).

Fixes #42
```

Dès que cette Pull Request est fusionnée dans `main`, l'issue `#42` se ferme automatiquement — et
l'historique du dépôt conserve un lien bidirectionnel permanent entre le bogue signalé et le commit
qui l'a réellement corrigé, consultable des mois plus tard sans avoir à chercher dans des messages
externes au dépôt.

<details>
<summary>🤔 Testez-vous</summary>

Une personne étudiante ouvre une Pull Request avec la description `Fixes #17`, mais cible par
erreur la branche `develop` plutôt que la branche par défaut du dépôt (`main`). La Pull Request est
fusionnée. L'issue `#17` se ferme-t-elle automatiquement ? Pourquoi ?

**Réponse** : non. Selon la documentation GitHub, les mots-clés de fermeture automatique ne sont
interprétés que lorsque la Pull Request cible la branche **par défaut** du dépôt. Comme cette Pull
Request cible `develop` (une branche différente), le mot-clé `Fixes #17` est ignoré : aucun lien
n'est créé, et fusionner la PR n'a aucun effet sur l'issue `#17`, qui doit alors être fermée
manuellement.
</details>

---

## 📚 Références

- [GitHub Docs — About issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/learning-about-issues/about-issues)
  — rôle général des issues et de leurs métadonnées.
- [GitHub Docs — Managing labels](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels)
  — étiquettes par défaut d'un nouveau dépôt.
- [GitHub Docs — About milestones](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/about-milestones)
  — regroupement d'issues par échéance commune.
- [GitHub Docs — Linking a pull request to an issue](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue)
  — mots-clés de fermeture automatique et leur limite (branche par défaut uniquement).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Distribution de mises à jour]({{% relref "/semaine-10/exercice-distribution-maj" %}})
pour appliquer ces notions à des situations concrètes.
{{% /notice %}}
