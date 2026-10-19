+++
title = "Gestion sémantique de version (SemVer)"
weight = 1
draft = false
+++

Tout ce qui suit est fondé sur la spécification officielle **Semantic Versioning 2.0.0**, publiée
par Tom Preston-Werner — voir la [version française intégrale](https://semver.org/lang/fr/)
et la [version originale, faisant foi](https://semver.org/). Les citations ci-dessous reprennent
la numérotation des règles de la spécification.

---

## 1️⃣ Le problème que SemVer résout

La spécification introduit le problème ainsi (traduction de la version officielle) :

> « Dans le monde de la gestion des logiciels, il existe un endroit redouté appelé « l'enfer des
> dépendances ». [...] Si les règles de dépendance sont trop strictes, vous risquez de verrouiller
> vos versions [...]. Si les règles de dépendances sont trop lâches, vous allez inévitablement
> subir la promiscuité de version. »
> — [semver.org/lang/fr](https://semver.org/lang/fr/)

Un numéro de version n'est donc pas qu'une étiquette cosmétique : c'est un **signal** envoyé à
quiconque dépend de votre logiciel (ou de votre API), qui lui permet de décider s'il peut mettre à
jour sans risque ou non.

---

## 2️⃣ Le format MAJEUR.MINEUR.CORRECTIF

La spécification résume les trois règles essentielles ainsi :

> « Étant donné un numéro de version MAJEUR.MINEUR.CORRECTIF, il faut incrémenter :
> 1. le numéro de version MAJEUR quand il y a des changements non rétrocompatibles,
> 2. le numéro de version MINEUR quand il y a des ajouts de fonctionnalités rétrocompatibles,
> 3. le numéro de version de CORRECTIF quand il y a des corrections d'anomalies rétrocompatibles. »
> — [semver.org/lang/fr, règles 6-8](https://semver.org/lang/fr/)

| Partie | Incrémentée quand… | Remet à zéro |
|---|---|---|
| **MAJEUR** (X) | Un changement **non rétrocompatible** est introduit dans l'API publique (règle 8) | MINEUR et CORRECTIF |
| **MINEUR** (Y) | Une nouvelle fonctionnalité **rétrocompatible** est ajoutée; DOIT aussi être incrémenté si une fonctionnalité existante est marquée obsolète (règle 7) | CORRECTIF |
| **CORRECTIF** (Z) | Seules des corrections de bogues **rétrocompatibles** sont introduites (règle 6) | — |

> 💡 La règle 3 de la spécification précise qu'une fois un numéro de version publié, « le contenu
> de sa version NE DOIT PAS être modifié » — toute correction, même minime, exige un **nouveau**
> numéro de version, jamais une republication silencieuse sous le même numéro.

> ⚠️ La règle 4 précise qu'un numéro de version majeure à **zéro** (`0.y.z`) correspond à du
> développement initial : « Tout PEUT être modifié à tout moment. L'API publique NE DEVRAIT PAS
> être considérée comme stable. » SemVer ne garantit donc une stabilité d'API qu'à partir de la
> version `1.0.0` (règle 5).

---

## 3️⃣ Analyser une version réelle : `4.0.2`

Le `pom.xml` du projet utilisé dans ce cours déclare :

```xml
<artifactId>spring-petclinic-rest</artifactId>
<version>4.0.2</version>
```

Décomposons ce numéro selon le format de la spécification (règles 6-8, voir section 2️⃣ ci-dessus) :

| Segment | Valeur | Signification selon SemVer |
|---|---|---|
| MAJEUR | `4` | 4ᵉ version majeure de ce composant |
| MINEUR | `0` | Aucune fonctionnalité additive publiée depuis `4.0.0` |
| CORRECTIF | `2` | Deux corrections rétrocompatibles publiées depuis `4.0.0` |

Si une équipe souhaitait signaler que cette version n'est pas encore stable (par exemple une
version de test interne avant publication), la règle 9 de la spécification permet d'ajouter un
**identifiant de pré-publication** après un trait d'union, par exemple `4.0.2-beta` :

> « Un numéro de version de pré-livraison PEUT être désigné en ajoutant un trait d'union et une
> série d'identifiants [...] séparés par des points après la version de correctif. Les identifiants
> DOIVENT être composés uniquement de caractères alphanumériques ASCII et de traits d'union. »
> — [semver.org/lang/fr, règle 9](https://semver.org/lang/fr/)

> « Une version de pré-livraison indique que la version n'est pas stable et ne satisfait pas
> forcément les exigences de compatibilité associées à une version normale. »
> — [semver.org/lang/fr, règle 9](https://semver.org/lang/fr/)

Le `pom.xml` actuel du projet (`4.0.2`, sans suffixe) n'utilise pas cette possibilité — c'est donc
une version normale, qui doit respecter pleinement les règles de compatibilité vues sur la page
suivante.

<details>
<summary>🤔 Testez-vous</summary>

Une équipe corrige un bogue qui faisait planter un endpoint existant, sans ajouter ni retirer
aucun champ de l'API. Le projet est actuellement en version `2.3.1`. Quelle doit être la
prochaine version, selon la règle 6 de la spécification ?

**Réponse** : `2.3.2`. La règle 6 impose d'incrémenter uniquement le numéro de CORRECTIF (Z)
lorsque « seules des corrections rétrocompatibles sont introduites » — le MAJEUR et le MINEUR ne
changent pas puisqu'aucune fonctionnalité n'a été ajoutée ni retirée de l'API publique.
</details>

---

## 4️⃣ CalVer : une alternative à SemVer, versionner par date

SemVer est répandu, mais ce n'est **ni le seul**, ni toujours le mieux adapté. Le *Versionnage
Calendaire* (**CalVer**) est une alternative réelle, utilisée par de grands projets :

> « CalVer est une convention de versionnage basée sur le calendrier de publication de votre
> projet [...]. Cette approche basée sur le temps s'appelle le Versionnage Calendaire, ou **CalVer**
> en abrégé. »
> — [calver.org](https://calver.org/)

Contrairement à SemVer, où chaque segment a une **signification** précise (compatibilité), CalVer
encode simplement la **date** de publication. Des projets réels, cités par calver.org, illustrent
différents découpages :

| Format | Exemple réel | Projet |
|---|---|---|
| `YY.0M` (année courte . mois) | `26.04` | Ubuntu |
| `YYYY.MINOR` (année . numéro séquentiel) | `2026.2` | JetBrains, Kali Linux |
| `YYYY.0M.0D` (année . mois . jour) | `2026.08.19` | yt-dlp |

> 💡 CalVer convient bien à des projets publiés à un **rythme régulier et prévisible** (ex. deux
> fois par année pour Ubuntu), où la date de sortie est elle-même une information utile pour
> l'utilisateur. Il convient moins bien à une API dont la compatibilité doit être communiquée
> explicitement à chaque changement — ce que SemVer fait par construction, mais pas CalVer.

---

## 📚 Références

- Tom Preston-Werner — [Semantic Versioning 2.0.0, version française intégrale](https://semver.org/lang/fr/).
- Tom Preston-Werner — [Semantic Versioning 2.0.0, version originale (faisant foi)](https://semver.org/).
- [calver.org](https://calver.org/) — Calendar Versioning.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Tags Git et publications (Releases) GitHub]({{% relref "/semaine-8/tags-releases-github" %}})
pour voir comment un numéro de version se transforme en tag Git, puis en Release GitHub automatisée.
{{% /notice %}}
