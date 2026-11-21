+++
title = "Exercice : distribution de mises à jour"
weight = 3
draft = false
+++

## 🎯 Objectif

Appliquer les concepts vus dans
[Distribution de mises à jour et sécurité]({{% relref "/semaine-10/distribution-maj-securite" %}})
et [Gestion des bogues avec GitHub Issues]({{% relref "/semaine-10/gestion-bogues-github-issues" %}})
à des situations concrètes.

---

## 🧩 Partie 1 — Objets connectés et rétroportage

Une équipe maintient un capteur de température connecté, déployé chez des centaines de clients,
dont le micrologiciel est actuellement en version **1.4** (stable, en production) pendant que la
version **2.0** (avec un nouveau protocole réseau incompatible) est en développement actif. Une
faille de sécurité critique est découverte et corrigée dans la base de code destinée à devenir
2.0.

1. Pourquoi l'équipe ne peut-elle pas simplement dire à ses clients « installez la 2.0 » pour
   corriger cette faille ?
2. Quelle opération Git, vue dès la semaine 2, permet concrètement de rétroporter ce correctif vers
   une branche `release-1.4` ? Quelle condition, tirée des règles officielles du noyau Linux
   (section 3️⃣ des notes), ce correctif doit-il respecter avant d'être rétroporté ?
3. Le micrologiciel 1.4.1 (corrigé) est signé et envoyé par liaison radio (OTA) à un capteur. Une
   coupure de courant survient pendant l'installation. En vous basant sur le mécanisme des mises à
   jour A/B (Android), que devrait-il se passer pour que le capteur reste utilisable ?

---

## 🧩 Partie 2 — Mises à jour signées

1. Expliquez, dans vos mots, ce qu'une signature numérique garantit à un appareil qui reçoit une
   mise à jour — et ce qu'elle **ne garantit pas** à elle seule (appuyez-vous sur l'exemple du
   retour en arrière, section 4️⃣ des notes).
2. Un attaquant parvient à compromettre un miroir de paquets Debian et y dépose une version
   modifiée d'un paquet existant, sans toucher au fichier `Release` signé par l'équipe Debian. Que
   se passe-t-il lorsqu'un utilisateur exécute `apt-get install` sur ce paquet modifié ? Appuyez
   votre réponse sur le mécanisme décrit dans `SecureApt`.

---

## 🧩 Partie 3 — GitHub Issues

Une équipe utilise GitHub Issues pour suivre les bogues trouvés par SonarCloud et Trivy
(semaine 9). Une Pull Request destinée à corriger l'issue `#58` est ouverte avec la description
suivante :

```markdown
Corrige la fuite de ressource signalée par SonarCloud dans `FileExporter`.

Closes #58
```

1. Cette Pull Request cible la branche `hotfix-securite`, qui sera plus tard fusionnée
   manuellement dans `main`. L'issue `#58` se fermera-t-elle automatiquement au moment de la
   fusion de cette Pull Request ? Justifiez à l'aide de la documentation GitHub citée dans les
   notes.
2. Proposez une étiquette (*label*) et une justification pour chacune des trois situations de la
   Partie 1 de l'[exercice de la semaine 9]({{% relref "/semaine-9/exercice-qualite-securite" %}})
   (injection SQL, duplication, division par zéro), en vous basant sur les étiquettes par défaut de
   GitHub.

---

<details>
<summary>❓ Corrigé — Partie 1</summary>

1. La version 2.0 change le protocole réseau de façon incompatible : migrer imposerait aux clients
   une mise à jour majeure (nouvelle intégration, nouveaux tests de leur côté) simplement pour
   obtenir un correctif de sécurité, ce qu'une grande partie d'entre eux refuserait ou ne pourrait
   pas faire à court terme.
2. `git cherry-pick` du commit contenant le correctif, appliqué sur la branche `release-1.4`. Selon
   les règles du noyau Linux, ce correctif doit déjà exister (ou un correctif équivalent) dans la
   branche la plus récente **avant** d'être rétroporté — on ne corrige jamais une faille
   uniquement dans une ancienne branche pendant qu'elle resterait présente dans la version actuelle.
3. Le capteur doit redémarrer automatiquement sur son ancienne partition (toujours intacte et
   fonctionnelle), rester pleinement utilisable avec le micrologiciel 1.4 précédent, et pouvoir
   retenter la mise à jour vers 1.4.1 plus tard — sans jamais rester bloqué dans un état
   intermédiaire inutilisable.
</details>

<details>
<summary>❓ Corrigé — Partie 2</summary>

1. Une signature numérique garantit que le fichier reçu provient bien du détenteur de la clé
   privée correspondante, et qu'il n'a pas été modifié depuis sa signature. Elle ne garantit **pas**
   qu'il s'agit de la version la **plus récente** : une ancienne version légitimement signée, mais
   contenant une faille depuis corrigée, reste parfaitement valide du point de vue de la seule
   signature — d'où la nécessité d'une protection contre le retour en arrière, distincte de la
   vérification de signature elle-même.
2. `apt-get install` vérifie que la somme de contrôle du paquet téléchargé correspond à celle
   listée dans le fichier `Release` signé. Le fichier `Release` n'ayant pas été modifié par
   l'attaquant, il contient toujours la somme de contrôle du paquet **original**. La somme de
   contrôle du paquet modifié par l'attaquant ne correspondra donc pas à celle attendue, et
   `apt-get` refusera d'installer le paquet falsifié.
</details>

<details>
<summary>❓ Corrigé — Partie 3</summary>

1. Non. La documentation GitHub précise que les mots-clés de fermeture automatique ne sont
   interprétés que lorsque la Pull Request cible la branche **par défaut** du dépôt. Comme cette
   Pull Request cible `hotfix-securite` et non `main`, `Closes #58` est ignoré; l'issue `#58` devra
   être fermée manuellement, même après la fusion ultérieure de `hotfix-securite` dans `main`.
2. Exemple de réponses acceptables : l'injection SQL correspond à l'étiquette `bug` (comportement
   non voulu exploitable), éventuellement combinée à une étiquette personnalisée comme
   `sécurité`/`priorité-haute` vu sa gravité; la duplication correspond plutôt à `enhancement` ou à
   une étiquette personnalisée comme `dette-technique`, puisqu'il ne s'agit ni d'un bogue ni d'une
   faille; la division par zéro correspond à `bug`, un défaut concret qui provoque un comportement
   incorrect dans un cas précis.
</details>

---

## ✅ Auto-vérification

- [ ] Je peux expliquer pourquoi la distribution d'une mise à jour est un problème distinct de sa
      construction, en particulier pour un objet connecté.
- [ ] Je peux relier le rétroportage de correctifs à une opération Git concrète déjà vue au cours
      (`cherry-pick`).
- [ ] Je peux expliquer ce qu'une signature numérique garantit, et ce qu'elle ne garantit pas sans
      protection contre le retour en arrière.
- [ ] Je sais dans quelles conditions un mot-clé comme `Closes #58` ferme automatiquement une issue
      GitHub, et dans quelles conditions il est ignoré.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[activité pratique]({{% relref "/semaine-10/activite-pratique" %}}) pour mettre en
place un vrai suivi de bogues outillé sur votre application legacy.
{{% /notice %}}
