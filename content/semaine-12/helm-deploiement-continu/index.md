+++
title = "Helm et déploiement sans interruption : charts, values.yaml, rolling update"
weight = 2
draft = false
+++

Tout ce qui suit est fondé sur la documentation officielle **Helm** (`helm.sh/docs`) et
**Kubernetes** (`kubernetes.io/docs`), appliquée au Deployment/Service `petclinic` des semaines
11-12.

---

## 1️⃣ Le problème que Helm résout : la duplication de YAML

Au fil des semaines 11 et 12, vous avez écrit, à la main, un `Deployment`, un `Service`, un
`ConfigMap`, un `Secret` et un `Ingress` — cinq fichiers YAML distincts pour une seule
application. Multipliez ça par chaque équipe, chaque environnement (développement, production), et
chaque mise à jour : vous copiez-collez le même YAML en ne changeant qu'un tag d'image ou une URL.

> 💡 **Le lien avec le réusinage (semaine 6)** : un `values.yaml` dupliqué entre deux équipes, où
> seule une URL ou un tag d'image change, est exactement le même problème de **duplication de
> code** identifié par SonarQube — sauf qu'ici, ce n'est pas du Java qui est dupliqué, mais de la
> configuration YAML. **Helm** répond à ce problème par un mécanisme équivalent à un
> *Extract Method/Template* : on factorise la structure commune dans des **templates**, et on
> n'exprime que les valeurs qui varient dans un fichier `values.yaml` séparé.

La documentation officielle Helm décrit ainsi le format qu'il utilise :

> « Helm uses a packaging format called _charts_. A chart is a collection of files that describe a
> related set of Kubernetes resources. »
> — [Helm — Charts](https://helm.sh/docs/topics/charts/)

### Structure d'un chart

```text
petclinic/
├── Chart.yaml          # Informations sur le chart (nom, version)
├── values.yaml         # Valeurs de configuration par défaut
├── templates/          # Manifestes Kubernetes paramétrés
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── secret.yaml
```

> « A chart is organized as a collection of files inside of a directory. [...] `Chart.yaml`: A
> YAML file containing information about the chart. `values.yaml`: The default configuration
> values for this chart. `templates/`: A directory of templates that, when combined with values,
> will generate valid Kubernetes manifest files. »
> — [Helm — Charts, The Chart File Structure](https://helm.sh/docs/topics/charts/)

Un `values.yaml` minimal pour `petclinic` :

```yaml
image:
  repository: ghcr.io/<votre-compte>/petclinic
  tag: latest
replicaCount: 2
service:
  type: LoadBalancer
  port: 9966
```

Et le template de Deployment correspondant, qui **référence** ces valeurs plutôt que de les
coder en dur (syntaxe `{{ .Values.<clé> }}`) :

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: petclinic
  template:
    metadata:
      labels:
        app: petclinic
    spec:
      containers:
        - name: petclinic
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 9966
```

> ⚠️ C'est exactement ce mécanisme qui permettra, en section 3️⃣, de changer **uniquement** le
> tag d'image lors d'une mise à jour — sans retoucher ni dupliquer le reste du Deployment.

Déployer ce chart pour la première fois (`helm install`), puis vérifier :

```shell
helm install petclinic ./petclinic
helm status petclinic
```

> « To install a new package, use the `helm install` command. At its simplest, it takes two
> arguments: A release name that you pick, and the name of the chart you want to install. »
> — [Helm — Using Helm](https://helm.sh/docs/intro/using_helm/)

---

## 2️⃣ Rolling update : la stratégie de déploiement par défaut

Une fois l'application en production, la vraie question devient : **comment la mettre à jour sans
couper le service ?** Par défaut, un Deployment Kubernetes utilise la stratégie **RollingUpdate** :

> « Deployment ensures that only a certain number of Pods are down while they are being updated.
> By default, it ensures that at least 75% of the desired number of Pods are up (25% max
> unavailable). Deployment also ensures that only a certain number of Pods are created above the
> desired number of Pods. By default, it ensures that at most 125% of the desired number of Pods
> are up (25% max surge). [...] it first creates a new Pod, then deletes an old Pod, and creates
> another new one. It does not kill old Pods until a sufficient number of new Pods have come up,
> and does not create new Pods until a sufficient number of old Pods have been killed. »
> — [Kubernetes — Deployments, Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)

Concrètement, avec `replicas: 2`, Kubernetes ne tue jamais les deux pods existants en même temps :
il démarre un nouveau pod, attend qu'il soit prêt, **puis seulement** retire un ancien pod — le
Service (semaine 11), dont le `selector` cible l'étiquette `app: petclinic` et non des pods
précis, continue de router le trafic vers les pods disponibles tout au long de l'opération.

Suivre un rollout en cours :

```shell
kubectl rollout status deployment/petclinic
# Waiting for rollout to finish: 1 out of 2 new replicas have been updated...
# deployment "petclinic" successfully rolled out
```

— [Kubernetes — Deployments, Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)

### Et les alternatives : blue-green et canary

Le rolling update n'est pas la seule stratégie possible — deux autres sont couramment citées,
même si **seul le rolling update est couvert en profondeur dans ce cours** :

- **Blue-green** : deux environnements complets et identiques tournent en parallèle (« blue » =
  version actuelle, « green » = nouvelle version); le trafic bascule d'un coup de l'un à l'autre
  une fois la nouvelle version validée. Avantage : retour en arrière instantané (on rebascule le
  trafic); coût : deux fois plus de ressources tournent en même temps.
- **Canary** : la nouvelle version est déployée à une **petite fraction** du trafic réel (ex. 5 %)
  avant d'être généralisée, pour détecter un problème sur un échantillon limité d'utilisateurs
  réels avant un déploiement complet.

> 💡 Ces deux stratégies ne sont pas une fonctionnalité native d'un simple Deployment Kubernetes —
> elles demandent généralement un outillage additionnel (deux Deployments distincts et un bascule
> manuel/scripté pour le blue-green; un contrôleur d'Ingress ou un service mesh pour le canary).
> Retenez surtout la distinction : le rolling update graduel que vous allez pratiquer est le plus
> simple des trois à mettre en œuvre, et c'est pour ça qu'il est le comportement **par défaut**
> de Kubernetes.

---

## 3️⃣ Déployer une mise à jour avec `helm upgrade`

Pour appliquer une nouvelle version de `petclinic` (par exemple, un nouveau tag d'image construit
par votre pipeline CI/CD des semaines 5 à 9), on ne réécrit **aucun** fichier YAML à la main : on
ne change que la valeur du tag, via `--set` :

```shell
helm upgrade petclinic ./petclinic --set image.tag=<nouveau-sha-ou-tag>
```

> « This command upgrades a release to a new version of a chart. [...] To override values in a
> chart, use either the '--values' flag and pass in a file or use the '--set' flag and pass
> configuration from the command line. »
> — [Helm — helm upgrade](https://helm.sh/docs/helm/helm_upgrade/)

Cette commande déclenche exactement le rolling update décrit en section 2️⃣ : Helm met à jour le
Deployment (nouveau tag d'image dans `spec.template`), et c'est **Kubernetes**, pas Helm, qui
orchestre ensuite le remplacement progressif des pods.

### Revenir en arrière automatiquement si l'upgrade échoue

> ⚠️ La documentation Helm 3 (la version la plus largement déployée en entreprise à l'heure
> actuelle) propose le drapeau `--atomic` : si l'upgrade échoue (ex. nouvelle image qui ne démarre
> jamais), Helm annule automatiquement les changements et restaure la révision précédente — sans
> intervention manuelle. La documentation **Helm 4** (la plus récente, publiée sur
> [helm.sh/docs](https://helm.sh/docs/)) a renommé ce comportement en un drapeau plus explicite :
>
> > « `--rollback-on-failure` — if set, Helm will rollback the upgrade to previous success release
> > upon failure. »
> > — [Helm — helm upgrade, Options](https://helm.sh/docs/helm/helm_upgrade/)
>
> Vérifiez donc, avec `helm version`, quelle version de Helm votre cluster utilise avant de choisir
> entre `--atomic` (Helm 3) et `--rollback-on-failure` (Helm 4) — le principe (annuler
> automatiquement un déploiement raté) reste le même dans les deux cas.

```shell
# Helm 3
helm upgrade petclinic ./petclinic --set image.tag=<nouveau-tag> --atomic

# Helm 4
helm upgrade petclinic ./petclinic --set image.tag=<nouveau-tag> --rollback-on-failure
```

Et pour revenir manuellement à la révision précédente à tout moment :

```shell
helm rollback petclinic
```

---

<details>
<summary>🤔 Testez-vous</summary>

**Question** : Votre équipe exécute `helm upgrade petclinic ./petclinic --set image.tag=v2`, mais
la nouvelle image `v2` contient un bogue qui empêche l'application de démarrer. Sans `--atomic`
(ou `--rollback-on-failure`), que se passe-t-il concrètement pour vos utilisateurs pendant que
Kubernetes tente le rolling update ?

**Réponse** : Kubernetes tente de démarrer un nouveau pod avec l'image `v2`; comme il ne devient
jamais « prêt » (il plante au démarrage), la stratégie RollingUpdate ne retire **jamais** les
anciens pods fonctionnels (« it does not kill old Pods until a sufficient number of new Pods have
come up » — [Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)).
Le service reste donc disponible via les anciens pods, mais le rollout reste **bloqué** en attente
indéfiniment — c'est précisément ce que `--atomic`/`--rollback-on-failure` évite, en détectant
l'échec et en annulant l'upgrade automatiquement plutôt que de laisser l'équipe découvrir le
problème plus tard.

</details>

---

## 📚 Références

- [Helm — Charts](https://helm.sh/docs/topics/charts/)
- [Helm — Using Helm](https://helm.sh/docs/intro/using_helm/)
- [Helm — helm upgrade](https://helm.sh/docs/helm/helm_upgrade/)
- [Kubernetes — Deployments, Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Kubernetes et Helm]({{% relref "/semaine-12/exercice-kubernetes-helm" %}})
pour valider votre compréhension, avant de passer au **TP4**, votre [activité pratique]({{% relref "/semaine-12/activite-pratique" %}}).
{{% /notice %}}
