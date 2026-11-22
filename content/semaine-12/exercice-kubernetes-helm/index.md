+++
title = "Exercice : Kubernetes 2/2 et Helm"
weight = 3
draft = false
+++

## 🎯 Objectif

Appliquer les concepts vus dans
[ConfigMap, Secrets, logs, Ingress]({{% relref "/semaine-12/configuration-kubernetes" %}}) et
[Helm et déploiement sans interruption]({{% relref "/semaine-12/helm-deploiement-continu" %}}).

---

## 🧩 Partie 1 — Lire un `values.yaml` et un `Chart.yaml`

```yaml
# Chart.yaml
apiVersion: v2
name: petclinic
version: 1.3.0
appVersion: "4.0.2"
```

```yaml
# values.yaml
image:
  repository: ghcr.io/equipe-07/petclinic
  tag: v4.0.2
replicaCount: 2
service:
  type: LoadBalancer
  port: 9966
```

1. Quelle est la différence entre `version` et `appVersion` dans `Chart.yaml` ? Si votre équipe
   publie une nouvelle image `petclinic:v4.0.3` sans changer la structure des manifestes du chart
   lui-même, quel champ devrait changer : `version`, `appVersion`, ou les deux ?
2. Un coéquipier modifie directement `image.tag: v4.0.3` dans le fichier `values.yaml`, commit le
   changement, puis exécute `helm install petclinic ./petclinic`. Qu'est-ce qui ne va pas avec
   cette commande, étant donné qu'une version précédente de `petclinic` tourne déjà dans le
   cluster ?
3. Donnez la commande corrigée pour appliquer ce changement de tag à un déploiement déjà existant,
   en vous assurant qu'un échec de déploiement soit automatiquement annulé.

---

## 🧩 Partie 2 — ConfigMap ou Secret ?

Pour chacune des données suivantes utilisées par `petclinic`, indiquez s'il faut la stocker dans un
**ConfigMap** ou un **Secret**, en justifiant à l'aide de
[ConfigMap, Secrets, logs, Ingress]({{% relref "/semaine-12/configuration-kubernetes" %}}) :

1. Le nom du profil Spring actif (`SPRING_PROFILES_ACTIVE=h2`).
2. Le mot de passe de connexion à la base de données (si votre profil en exige un).
3. Le jeton d'API utilisé pour appeler un service tiers de géocodage d'adresses.
4. Le niveau de verbosité des logs de l'application (`INFO`, `DEBUG`).

Un coéquipier affirme que stocker le mot de passe dans un Secret suffit à le protéger contre
n'importe qui ayant un accès `kubectl get secret` sur le namespace. Est-ce exact ?

---

## 🧩 Partie 3 — Rolling update interrompu

Votre équipe exécute :

```shell
helm upgrade petclinic ./petclinic --set image.tag=v4.0.3
```

La nouvelle image `v4.0.3` contient un bogue qui fait planter l'application dès son démarrage
(elle ne devient jamais `Ready`).

1. D'après la stratégie `RollingUpdate` par défaut d'un Deployment, les pods de la version
   `v4.0.2` encore fonctionnels sont-ils retirés avant que les nouveaux pods `v4.0.3` soient prêts ?
   Justifiez.
2. Le Service reste-t-il disponible pour les utilisateurs pendant cet incident ? Pourquoi ?
3. Qu'auriez-vous pu ajouter à la commande `helm upgrade` initiale pour que Helm annule
   automatiquement ce déploiement raté, sans intervention manuelle ?

---

<details>
<summary>❓ Corrigé — Partie 1</summary>

1. `version` est la version du **chart lui-même** (sa structure de templates/values) et doit
   changer chaque fois que le chart change, selon SemVer; `appVersion` reflète la version de
   l'**application** empaquetée (ici `4.0.2`), et n'a pas besoin de suivre SemVer. Si seule l'image
   change (même structure de chart), c'est `appVersion` (et la valeur `image.tag` dans
   `values.yaml`) qui doit être mise à jour — `version` ne change que si le chart lui-même (ses
   templates) est modifié.
2. `helm install` sert à créer une **nouvelle** release; si `petclinic` tourne déjà dans le
   cluster, cette commande échouera (nom de release déjà pris) ou créera une collision de
   ressources. Il fallait utiliser `helm upgrade` pour mettre à jour une release existante.
3. `helm upgrade petclinic ./petclinic --set image.tag=v4.0.3 --atomic` (Helm 3) ou
   `--rollback-on-failure` (Helm 4) — selon la version de Helm installée sur le poste/pipeline de
   l'équipe.
</details>

<details>
<summary>❓ Corrigé — Partie 2</summary>

1. **ConfigMap** — une donnée non confidentielle.
2. **Secret** — une donnée sensible.
3. **Secret** — un jeton d'authentification est une donnée sensible, au même titre qu'un mot de
   passe.
4. **ConfigMap** — une donnée non confidentielle.

Non, ce n'est **pas exact** : un Secret Kubernetes n'est, par défaut, pas chiffré dans etcd — ses
valeurs ne sont qu'encodées en base64, une opération réversible sans aucune clé. Quiconque ayant
accès à l'API du cluster (via `kubectl get secret -o yaml`, par exemple) peut décoder la valeur
immédiatement. Une protection réelle exige d'activer le chiffrement au repos (*Encryption at
Rest*) et des règles RBAC restreignant qui peut lire les Secrets.
</details>

<details>
<summary>❓ Corrigé — Partie 3</summary>

1. Non — la documentation officielle précise que le Deployment « does not kill old Pods until a
   sufficient number of new Pods have come up ». Comme les nouveaux pods `v4.0.3` ne deviennent
   jamais `Ready`, les anciens pods `v4.0.2` encore fonctionnels ne sont jamais retirés.
2. Oui, le Service reste disponible : son `selector` continue de cibler les pods `v4.0.2` toujours
   en fonction, pendant que Kubernetes attend que les nouveaux pods deviennent prêts (ce qui
   n'arrivera jamais ici).
3. `--atomic` (Helm 3) ou `--rollback-on-failure` (Helm 4) : Helm aurait détecté que le rollout ne
   se termine jamais avec succès et aurait automatiquement restauré la révision précédente, sans
   qu'un humain ait à intervenir pour faire un `helm rollback` manuel.
</details>

---

## ✅ Auto-vérification

- [ ] Je peux expliquer la différence entre `version` et `appVersion` dans un `Chart.yaml`.
- [ ] Je sais distinguer quand utiliser un ConfigMap plutôt qu'un Secret, et inversement.
- [ ] Je peux expliquer pourquoi un Secret Kubernetes n'est pas, par défaut, une donnée chiffrée.
- [ ] Je peux expliquer comment un rolling update protège la disponibilité du service même quand
      la nouvelle version échoue, et je connais le drapeau qui déclenche un rollback automatique.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[activité pratique]({{% relref "/semaine-12/activite-pratique" %}}) pour déployer réellement
une mise à jour de votre application, via Helm, sans interruption de service.
{{% /notice %}}
