+++
title = "Distribution de mises à jour et sécurité"
weight = 1
draft = false
+++

Tout ce qui suit est fondé sur des sources officielles : la documentation du noyau Linux
(`kernel.org`), l'alerte CISA sur le botnet Mirai, le wiki officiel de Debian sur `apt-secure`, la
documentation Android Open Source Project (AOSP) sur l'Android Verified Boot, et la spécification
*The Update Framework* (TUF), un projet de la Cloud Native Computing Foundation (CNCF).

---

## 1️⃣ Construire n'est pas distribuer

Les semaines précédentes ont toutes porté sur la **construction** d'un changement : écrire du
code (semaine 6), le tester (semaines 4-5), le conteneuriser et le publier dans un registre
(semaine 7), le versionner (semaine 8), vérifier sa qualité et sa sécurité (semaine 9). Une fois
une image publiée dans un registre de conteneurs, un problème distinct se pose : comment cette
mise à jour **atteint-elle** réellement chaque instance en production ?

Pour un service web classique (comme l'application legacy du cours, déployée sur Kubernetes aux
semaines 11-12), la réponse est relativement simple : l'équipe contrôle l'infrastructure et déploie
elle-même la nouvelle image, au moment de son choix. Trois situations rendent ce problème
nettement plus difficile, et sont au cœur de cette semaine :

- **Les objets connectés (IoT)** : l'appareil n'est pas sous le contrôle direct de l'équipe qui l'a
  construit — il est chez un client, parfois difficile d'accès, avec une connectivité limitée.
- **Le rétroportage de correctifs** : tous les utilisateurs ne peuvent pas (ou ne veulent pas)
  migrer vers la toute dernière version dès qu'un correctif sort.
- **Les mises à jour signées** : dans les deux cas précédents, la mise à jour transite par un canal
  (réseau, support physique, miroir de paquets) qui n'est pas nécessairement de confiance — il faut
  pouvoir vérifier qu'elle n'a pas été altérée en chemin.

---

## 2️⃣ Objets connectés : des contraintes qui changent tout

Un objet connecté (capteur, caméra, routeur domestique, appareil médical, véhicule) diffère d'un
serveur applicatif sur plusieurs points qui compliquent directement la distribution de mises à
jour :

| Contrainte | Conséquence sur la mise à jour |
|---|---|
| Pas d'écran, pas de clavier | Aucune interaction humaine possible pour confirmer ou déboguer une mise à jour |
| Connectivité intermittente ou limitée | Impossible de supposer une connexion fiable et à haut débit en tout temps |
| Parc déployé, difficile d'accès physiquement | Un appareil qui ne redémarre plus après une mise à jour ratée (*brick*) peut nécessiter un déplacement coûteux, voire un remplacement |
| Durée de vie très longue (parfois 10+ ans) | Le logiciel doit rester **sécuritaire** bien après la fin du support actif de ses dépendances |

### Le risque concret d'une absence de mise à jour

Ces contraintes ne sont pas seulement théoriques. En 2016, le botnet **Mirai** a exploité des
centaines de milliers d'objets connectés (routeurs domestiques, caméras réseau, enregistreurs
vidéo numériques) pour mener des attaques par déni de service parmi les plus importantes jamais
observées à l'époque :

> « Le logiciel malveillant Mirai scanne continuellement Internet à la recherche d'appareils IoT
> vulnérables, qui sont ensuite infectés et utilisés dans des attaques de botnet. Le bot Mirai
> utilise une courte liste de 62 noms d'utilisateur et mots de passe par défaut courants pour
> trouver des appareils vulnérables. Parce que de nombreux appareils IoT sont non sécurisés ou
> faiblement sécurisés, cette courte liste permet au bot d'accéder à des centaines de milliers
> d'appareils. »
> — [CISA, alerte TA16-288A — Heightened DDoS Threat Posed by Mirai and Other Botnets](https://www.cisa.gov/news-events/alerts/2016/10/14/heightened-ddos-threat-posed-mirai-and-other-botnets)

Le problème initial de Mirai est une mauvaise gestion des identifiants par défaut, pas directement
une absence de mise à jour. Mais le constat qui en a découlé pour l'industrie est direct : une
fois une vulnérabilité de ce type rendue publique, les appareils affectés sont restés vulnérables
aussi longtemps qu'**aucun mécanisme fiable** ne permettait de leur livrer un correctif — ni
notification à l'utilisateur, ni mise à jour automatique dans la plupart des cas. C'est précisément
ce scénario que les mécanismes des sections suivantes (mise à jour signée, rétroportage) cherchent
à rendre possible, et sûr.

### Une mise à jour qui ne doit jamais rendre l'appareil inutilisable

Au-delà de livrer la mise à jour, il faut garantir qu'une mise à jour **ratée** (coupure de courant
pendant l'écriture, image corrompue) ne transforme jamais l'appareil en presse-papier. Android,
qui gère un parc de plus d'un milliard d'appareils aux conditions réseau très variables, documente
sa solution : les **mises à jour A/B** (ou *seamless updates*), qui conservent toujours une
partition de démarrage fonctionnelle pendant l'installation d'une mise à jour :

> « Les mises à jour système A/B [...] garantissent qu'un système de démarrage fonctionnel reste
> sur le disque pendant une mise à jour par liaison radio (OTA, *over-the-air*). [...] Si une mise
> à jour OTA échoue à s'appliquer, l'utilisateur n'est pas affecté : il continue à utiliser
> l'ancien système d'exploitation, et peut retenter la mise à jour. Si une mise à jour OTA est
> appliquée mais échoue à démarrer, l'appareil redémarre dans l'ancienne partition et reste
> utilisable. »
> — [Android Open Source Project — A/B (Seamless) System Updates](https://source.android.com/docs/core/ota/ab)

Le principe est simple mais puissant : l'appareil possède **deux** copies (A et B) de son système.
La mise à jour s'installe entièrement sur la copie inactive pendant que l'appareil continue de
fonctionner normalement sur l'autre; ce n'est qu'au redémarrage suivant, une fois l'installation
confirmée réussie, que l'appareil bascule vers la nouvelle copie — avec un retour automatique à
l'ancienne copie en cas d'échec. Cette idée (toujours garder un état de secours fonctionnel)
reviendra sous une forme différente au moment du **rollback** d'un déploiement Kubernetes
(semaine 12).

---

## 3️⃣ Rétroportage de correctifs (*backporting*)

### Le problème : tout le monde ne peut pas migrer vers la dernière version

Imaginez une équipe qui maintient une version 2.4 de son logiciel, déjà déployée chez plusieurs
clients, pendant qu'une version 3.0 — avec de nouvelles fonctionnalités et des changements
d'API — est en développement actif. Une faille de sécurité est découverte et corrigée dans le code
**actuel** (la base destinée à devenir 3.0). Les clients sur la 2.4 ont besoin de ce correctif,
mais ne veulent pas (ou ne peuvent pas) migrer immédiatement vers 3.0, qui change leur intégration
et exige de nouveaux tests de leur côté.

Le **rétroportage** consiste précisément à réappliquer ce correctif, isolément, sur l'ancienne
version — sans y apporter les autres changements qui l'accompagnaient dans la version la plus
récente.

### Un exemple réel, documenté : le noyau Linux

Le noyau Linux maintient plusieurs branches **stables** en parallèle (une par version majeure
encore supportée) et publie un ensemble de règles très strictes sur ce qui peut y être rétroporté :

> « Il doit déjà exister dans la branche principale (*mainline*, upstream) de Linux, ou un correctif
> équivalent. [...] Il doit corriger un bogue réel qui dérange des utilisateurs, ou simplement
> ajouter un identifiant de périphérique. [...] Aucun correctif du type "cela pourrait être un
> problème..." comme une "condition de concurrence critique théorique", à moins qu'une explication
> de la façon dont le bogue peut être exploité ne soit également fournie. »
> — [kernel.org — Everything you ever wanted to know about Linux -stable releases](https://www.kernel.org/doc/html/latest/process/stable-kernel-rules.html)

Deux points structurent cette règle, directement transposables à n'importe quel projet du cours :

- **Le correctif doit déjà exister ailleurs** (dans la version la plus récente) avant d'être
  rétroporté — on ne corrige jamais un bogue uniquement dans une ancienne branche, pour éviter que
  la même faille persiste silencieusement dans la version actuelle.
- **Le correctif rétroporté doit rester minimal et ciblé** — la documentation précise qu'il « ne
  peut pas dépasser 100 lignes, avec contexte » : l'objectif est de corriger *un* problème précis,
  pas de faire converger l'ancienne branche vers la nouvelle.

Techniquement, rétroporter un correctif revient le plus souvent à appliquer, sur une ancienne
branche, le même commit déjà fusionné ailleurs — exactement l'opération vue à la semaine 2 :

```bash
# Le correctif a déjà été fusionné dans main (ex. commit a1b2c3d)
git switch release-2.4
git cherry-pick a1b2c3d
```

> 💡 **Le lien avec `cherry-pick`** (semaine 2) est direct : rétroporter un correctif, dans sa forme
> la plus simple, *est* un `cherry-pick` d'un commit déjà présent sur une autre branche, vers une
> branche plus ancienne. La difficulté en pratique vient du fait que le code environnant a souvent
> trop divergé entre les deux versions pour que le `cherry-pick` s'applique sans conflit — il faut
> alors réécrire manuellement un correctif équivalent, raison pour laquelle le noyau Linux prévoit
> explicitement cette possibilité (« Option 3 », un correctif équivalent plutôt qu'identique) dans
> sa documentation.

---

## 4️⃣ Mises à jour signées

### Le problème : faire confiance à un fichier reçu par le réseau

Qu'il s'agisse d'un paquet Debian téléchargé depuis un miroir, d'une mise à jour OTA reçue par un
objet connecté, ou d'une image Docker récupérée d'un registre (semaine 7), la mise à jour transite
par un canal que l'appareil ou la machine ne contrôle pas entièrement : un miroir compromis, une
attaque de l'intercepteur (*man-in-the-middle*), ou un registre piraté pourrait substituer un
fichier malveillant à l'original. La **signature numérique** répond à une question précise : *ce
fichier provient-il réellement de l'éditeur que je crois, et n'a-t-il pas été modifié depuis ?*

### Un exemple réel, documenté : la distribution sécurisée de paquets Debian

Depuis 2005, Debian signe cryptographiquement les métadonnées de son dépôt de paquets, un mécanisme
que le projet appelle `apt-secure` :

> « Debian utilise une cryptographie forte pour valider les paquets téléchargés. [...] Il est
> possible d'utiliser une clé privée pour signer un fichier, sans le chiffrer. Si une clé privée
> est utilisée pour signer un fichier, alors quiconque possède la clé publique peut vérifier que le
> fichier a été signé par cette clé. Quiconque ne possède pas la clé privée ne peut pas forger une
> telle signature. »
> — [wiki.debian.org — SecureApt](https://wiki.debian.org/SecureApt)

Concrètement, chaque dépôt Debian publie un fichier `Release` contenant les sommes de contrôle
(checksums) de tous les paquets qu'il propose, et ce fichier `Release` lui-même est signé (fichier
`Release.gpg`) avec la clé privée de l'équipe Debian. La commande `apt-get update` télécharge ces
métadonnées et vérifie la signature **avant** de faire confiance à quoi que ce soit; `apt-get
install` vérifie ensuite que la somme de contrôle du paquet téléchargé correspond bien à celle
listée (et donc implicitement signée) dans `Release`. Un attaquant qui compromettrait un simple
miroir ne pourrait donc pas substituer un paquet malveillant sans que la vérification de somme de
contrôle échoue.

### Un exemple réel, documenté : le démarrage vérifié d'Android

Le même principe (vérifier une signature avant de faire confiance à du code) s'applique, à un
niveau plus bas, directement au démarrage d'un appareil Android :

> « Le démarrage vérifié (*Verified Boot*) vise à garantir que tout le code exécuté provient d'une
> source de confiance (généralement les fabricants d'appareils), plutôt que d'un attaquant ou d'une
> corruption. Il établit une chaîne de confiance complète, à partir d'une racine de confiance
> protégée matériellement jusqu'au chargeur d'amorçage, jusqu'à la partition de démarrage et
> d'autres partitions vérifiées. »
> — [Android Open Source Project — Verified Boot](https://source.android.com/docs/security/features/verifiedboot)

La documentation ajoute un détail essentiel, directement lié au rétroportage (section précédente) :
le démarrage vérifié inclut une **protection contre le retour en arrière** (*rollback protection*),
qui empêche un attaquant de réinstaller volontairement une **ancienne** version du système — y
compris une version légitime, mais qui contiendrait une faille déjà corrigée depuis. Une signature
valide ne suffit donc pas à elle seule : encore faut-il vérifier que la version signée est bien la
plus récente connue, sans quoi un attaquant pourrait « mettre à jour » un appareil vers une version
ancienne et vulnérable, tout en conservant une signature parfaitement légitime.

### Généraliser le problème : The Update Framework (TUF)

Les deux exemples précédents (Debian, Android) résolvent chacun ce problème dans leur propre
écosystème. **The Update Framework (TUF)**, un projet *graduated* de la Cloud Native Computing
Foundation (CNCF — la même fondation qui héberge Kubernetes, vu aux semaines 11-12), propose un
cadre générique applicable à n'importe quel système de mise à jour logicielle :

> « The Update Framework (TUF) maintient la sécurité des systèmes de mise à jour logicielle, en
> offrant une protection même contre des attaquants qui compromettraient le dépôt ou les clés de
> signature. TUF fournit un cadre flexible et une spécification que les développeurs peuvent
> adopter dans n'importe quel système de mise à jour logicielle. »
> — [theupdateframework.io](https://theupdateframework.io/)

L'apport principal de TUF par rapport à une simple signature est de répondre à une question que la
signature seule ne couvre pas : *que se passe-t-il si la clé de signature elle-même est
compromise ?* TUF répond par une hiérarchie de **plusieurs rôles de clés** avec des responsabilités
distinctes (une clé « racine » rarement utilisée et très protégée, des clés « cibles » utilisées au
quotidien pour signer chaque version), de sorte que la compromission d'une seule clé ne suffise pas
à distribuer une mise à jour malveillante à l'insu de tous.

<details>
<summary>🤔 Testez-vous</summary>

Une équipe publie une mise à jour signée avec sa clé privée légitime pour un objet connecté. Un
attaquant intercepte la communication et renvoie à l'appareil une **ancienne** version du
logiciel — signée légitimement, à l'époque, par la même équipe — qui contient une faille déjà
corrigée depuis. La vérification de signature seule empêche-t-elle cette attaque ? Quel mécanisme,
vu dans cette page, y répond spécifiquement ?

**Réponse** : non, la vérification de signature seule ne suffit pas : la signature de l'ancienne
version est parfaitement valide, puisqu'elle a réellement été produite par l'équipe légitime. C'est
la **protection contre le retour en arrière** (*rollback protection*), comme celle décrite dans
Android Verified Boot, qui répond à ce scénario précis : l'appareil refuse d'installer une version
signée mais **plus ancienne** que la dernière version connue, peu importe la validité de sa
signature.
</details>

---

## 📚 Références

- [kernel.org — Everything you ever wanted to know about Linux -stable releases](https://www.kernel.org/doc/html/latest/process/stable-kernel-rules.html)
  — règles officielles de rétroportage de correctifs dans le noyau Linux.
- [CISA, alerte TA16-288A — Heightened DDoS Threat Posed by Mirai and Other Botnets](https://www.cisa.gov/news-events/alerts/2016/10/14/heightened-ddos-threat-posed-mirai-and-other-botnets)
  — cas réel documenté de risque lié aux objets connectés non sécurisés/non mis à jour.
- [Android Open Source Project — A/B (Seamless) System Updates](https://source.android.com/docs/core/ota/ab)
  — mécanisme de mise à jour à l'épreuve des échecs pour un parc d'appareils déployés.
- [Android Open Source Project — Verified Boot](https://source.android.com/docs/security/features/verifiedboot)
  — chaîne de confiance et protection contre le retour en arrière.
- [wiki.debian.org — SecureApt](https://wiki.debian.org/SecureApt)
  — signature cryptographique des dépôts de paquets Debian (`apt-secure`).
- [theupdateframework.io](https://theupdateframework.io/) — spécification générique de sécurisation
  des systèmes de mise à jour logicielle (projet CNCF).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Gestion des bogues avec GitHub Issues]({{% relref "/semaine-10/gestion-bogues-github-issues" %}})
pour voir comment les bogues qui motivent ces correctifs et ces rétroportages sont suivis et
communiqués au sein d'une équipe.
{{% /notice %}}
