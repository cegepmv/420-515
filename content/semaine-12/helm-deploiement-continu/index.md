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

> « Helm utilise un format d'empaquetage appelé _charts_. Un chart est un ensemble de fichiers qui
> décrivent un groupe de ressources Kubernetes liées entre elles. »
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

> « Un chart est organisé comme un ensemble de fichiers à l'intérieur d'un répertoire. [...]
> `Chart.yaml` : un fichier YAML contenant des informations sur le chart. `values.yaml` : les
> valeurs de configuration par défaut de ce chart. `templates/` : un répertoire de gabarits qui,
> combinés aux valeurs, génèreront des manifestes Kubernetes valides. »
> — [Helm — Charts, The Chart File Structure](https://helm.sh/docs/topics/charts/)

### Le fichier `Chart.yaml`

`Chart.yaml` est **obligatoire** pour tout chart — il identifie le chart lui-même (pas
l'application qu'il déploie). Un exemple minimal pour `petclinic` :

```yaml
# Chart.yaml
apiVersion: v2
name: petclinic
description: Chart Helm pour déployer l'application petclinic sur Kubernetes
type: application
version: 1.0.0
appVersion: "4.0.2"
```

Ligne par ligne, selon la documentation officielle Helm :

- **`apiVersion: v2`** — la version du format `Chart.yaml` compris par Helm. `v2` est la version
  courante (utilisée par Helm 3+); `v1` n'existe que pour les charts écrits pour Helm 2, une
  version obsolète.
- **`name: petclinic`** — le nom du chart. Par convention, il correspond au nom du répertoire qui
  le contient (`petclinic/Chart.yaml`).
- **`description`** — une phrase résumant ce que fait le chart, affichée par `helm search` et
  `helm show chart`. Champ optionnel, mais utile dès qu'un chart est partagé avec d'autres
  personnes.
- **`type: application`** — indique que ce chart déploie une application directement installable
  (par opposition à `library`, un chart réutilisé comme dépendance par d'autres charts, sans
  installation directe possible).
- **`version: 1.0.0`** — la version **du chart lui-même**, selon SemVer : elle doit changer
  chaque fois que la structure du chart change (ex. ajout d'un nouveau template, modification des
  clés de `values.yaml`) — **jamais** uniquement parce que l'image de l'application a changé.
- **`appVersion: "4.0.2"`** — la version de **l'application** empaquetée par ce chart (ici,
  `petclinic`). Contrairement à `version`, elle n'a pas besoin de suivre SemVer, et change
  typiquement à chaque nouvelle image déployée (ex. `4.0.3`) — **sans** nécessairement faire
  bouger `version`, puisque la structure du chart, elle, n'a pas changé. Notez les guillemets :
  Helm exige une chaîne de caractères ici, pour éviter qu'un numéro comme `4.0` soit interprété
  comme un nombre décimal par le parseur YAML.

> 💡 Cette distinction `version`/`appVersion` est exactement ce qui est testé à la partie 1 de
> l'[exercice]({{% relref "/semaine-12/exercice-kubernetes-helm" %}}) de cette semaine.

### Le fichier `values.yaml`

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

> « Pour installer un nouveau paquet, utilisez la commande `helm install`. Dans sa forme la plus
> simple, elle prend deux arguments : un nom de release que vous choisissez, et le nom du chart
> que vous voulez installer. »
> — [Helm — Using Helm](https://helm.sh/docs/intro/using_helm/)

---

## 2️⃣ Rolling update : la stratégie de déploiement par défaut

Une fois l'application en production, la vraie question devient : **comment la mettre à jour sans
couper le service ?** Par défaut, un Deployment Kubernetes utilise la stratégie **RollingUpdate** :

> « Un Deployment s'assure qu'un certain nombre de Pods seulement sont indisponibles pendant leur
> mise à jour. Par défaut, il garantit qu'au moins 75 % du nombre désiré de Pods sont
> opérationnels (25 % max d'indisponibilité). Le Deployment s'assure aussi qu'un certain nombre de
> Pods seulement sont créés au-delà du nombre désiré. Par défaut, il garantit qu'au plus 125 % du
> nombre désiré de Pods sont opérationnels (25 % max de surplus). [...] il crée d'abord un nouveau
> Pod, puis supprime un ancien Pod, et en crée un autre nouveau. Il ne détruit pas les anciens Pods
> tant qu'un nombre suffisant de nouveaux Pods ne sont pas arrivés, et ne crée pas de nouveaux Pods
> tant qu'un nombre suffisant d'anciens Pods n'ont pas été détruits. »
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

> « Cette commande met à niveau une release vers une nouvelle version d'un chart. [...] Pour
> remplacer des valeurs dans un chart, utilisez soit l'option « --values » en passant un fichier,
> soit l'option « --set » en passant la configuration directement depuis la ligne de commande. »
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
> > « `--rollback-on-failure` — si activé, Helm annule l'upgrade en revenant à la précédente
> > release réussie en cas d'échec. »
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

## 4️⃣ Automatiser le déploiement depuis le pipeline CI/CD

### Le problème : encore des commandes tapées à la main

Les sections 1️⃣ à 3️⃣ exigent encore qu'un humain tape `helm upgrade` après chaque nouvelle image —
exactement le problème que la **CI** (semaines 5 à 9) a déjà réglé pour la compilation, les tests
et la construction de l'image : une suite de gestes répétitifs, toujours les mêmes, qu'un oubli
humain finit par briser. La **livraison continue** (CD) prend le relais de la CI : une fois
l'image vérifiée et poussée sur GHCR (CI), c'est le pipeline lui-même — pas une personne — qui va
la chercher et la déploie sur le cluster. Dans ce cours, cette dernière étape se fait avec
`helm upgrade`, exactement la commande de la section 3️⃣, mais exécutée par GitHub Actions plutôt
que depuis votre terminal.

### Qui autorise GitHub à toucher à votre cluster Azure ?

Le pipeline s'exécute sur les machines de GitHub, pas sur votre poste : pour appliquer un
`helm upgrade` sur votre cluster AKS, il doit d'abord prouver à Azure qu'il en a le droit. Copier
un mot de passe permanent de votre compte Azure dans un secret GitHub *fonctionnerait*, mais ce
mot de passe pourrait fuir (capture d'écran, journal de pipeline trop bavard) et n'expire jamais
de lui-même — l'équivalent d'une copie de clé d'appartement laissée chez un tiers, pour toujours.

La méthode recommandée aujourd'hui est l'**OIDC** (*OpenID Connect*) : aucun mot de passe n'est
stocké dans GitHub. À chaque exécution, GitHub présente à Azure un badge temporaire, valable
seulement quelques minutes, prouvant « je suis ce dépôt précis, cette branche précise » — et Azure
ne l'accepte que parce que vous avez déclaré **à l'avance** (une **fédération**) que ce dépôt-là,
et aucun autre, peut se présenter ainsi. L'identité qui reçoit ce badge s'appelle un **principal de
service** (*service principal*), et le rôle qu'on lui donne détermine ce qu'il peut toucher — ici,
limité au **seul** groupe de ressources du cluster utilisé par votre équipe, jamais plus.

> ⚠️ **Qui fait cette configuration, une seule fois ?** Le cluster AKS utilisé pour l'activité pratique
> appartient au compte Azure **personnel** d'un membre de l'équipe (voir
> [Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}})) : c'est
> donc **ce membre précis**, et lui seul, qui peut créer cette autorisation sur son propre
> abonnement Azure, pour le dépôt GitHub de l'équipe. Les secrets qui en résultent (ci-dessous)
> sont ensuite stockés dans le dépôt de l'équipe, pas sur son compte personnel.

Mettre en place cette autorisation (à exécuter une seule fois, idéalement depuis le
[Azure Cloud Shell](https://shell.azure.com/), en remplaçant `rg-etu1234`, `app-github-etu1234`
et `MON-COMPTE/MON-DEPOT` par vos propres valeurs) :

```bash
RG=rg-etu1234
APP_NAME=app-github-etu1234
REPO="MON-COMPTE/MON-DEPOT"
SUB=$(az account show --query id -o tsv)
TENANT=$(az account show --query tenantId -o tsv)

# 1. Créer l'identité qu'utilisera GitHub
az ad app create --display-name $APP_NAME
APP_ID=$(az ad app list --display-name $APP_NAME --query "[0].appId" -o tsv)
az ad sp create --id $APP_ID

# 2. Lui donner le rôle Contributeur, UNIQUEMENT sur le groupe de ressources du cluster
az role assignment create \
  --assignee $APP_ID \
  --role Contributor \
  --scope /subscriptions/$SUB/resourceGroups/$RG

# 3. Déclarer la fédération : seul CE dépôt, branche main, peut se présenter avec cette identité
cat > federated.json <<EOF
{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:$REPO:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}
EOF
az ad app federated-credential create --id $APP_ID --parameters federated.json

# 4. Les trois valeurs à recopier dans GitHub (dépôt → Settings → Secrets and variables → Actions)
echo "AZURE_CLIENT_ID = $APP_ID"
echo "AZURE_TENANT_ID = $TENANT"
echo "AZURE_SUBSCRIPTION_ID = $SUB"
```

> 💡 Aucune de ces trois valeurs (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`)
> n'est un mot de passe — ce sont de simples identifiants, sans valeur si quelqu'un les lit sans
> avoir la fédération déclarée à l'étape 3. C'est **elle** qui protège réellement l'accès.

### Le job de déploiement, ajouté au pipeline CI

Ajoutez ce `job` au fichier de workflow déjà utilisé pour la CI (semaines 5 à 9), après le job qui
construit et pousse l'image sur GHCR :

```yaml
permissions:
  id-token: write   # requis pour s'authentifier à Azure par OIDC
  contents: read

jobs:
  # ... job existant build-and-push (semaines 5 à 9) ...

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Connexion à Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Viser le cluster AKS de l'équipe
        uses: azure/aks-set-context@v4
        with:
          resource-group: rg-etu1234
          cluster-name: aks-etu1234

      - name: Se placer dans le bon namespace
        run: kubectl config set-context --current --namespace=etu1234

      - name: Déployer la nouvelle image avec Helm
        run: |
          helm upgrade petclinic ./petclinic \
            --set image.tag=${{ github.sha }} \
            --atomic

      - name: Confirmer que le rollout a réussi
        run: kubectl rollout status deployment/petclinic
```

Trois détails à retenir : (1) `image.tag=${{ github.sha }}` réutilise **exactement** le tag déjà
construit et poussé par le job de CI — jamais `latest`, pour la même raison déjà vue à la
semaine 7 (traçabilité : on sait **exactement** quel commit tourne en production); (2) le cluster
doit être **démarré** (`az aks start`, section 6️⃣ de
[Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}})) pour
qu'un pipeline puisse y déployer — un cluster arrêté fait échouer ce job; (3) `--atomic` (ou
`kubectl rollout status` seul, selon votre choix) fait **échouer visiblement** le pipeline si le
déploiement ne se stabilise jamais, plutôt que d'afficher un faux succès sur une application
cassée.

> ⚠️ Un `Secret` Kubernetes créé **à la main** sur votre cluster (ex. `petclinic-db-secret`,
> section 2️⃣) ne passe **jamais** par ce pipeline : il reste créé une fois, manuellement, hors de
> Git — le job `deploy` ne fait que `helm upgrade`, qui ne touche pas aux Secrets déjà en place sur
> le cluster.

### Revenir en arrière après un déploiement automatisé raté

Le pipeline peut, lui aussi, déployer une version cassée. L'avantage d'un tag par SHA de commit et
de l'historique conservé par Helm, c'est qu'un retour en arrière prend quelques secondes, sans
toucher au cluster à la main :

```shell
helm rollback petclinic
kubectl rollout status deployment/petclinic
```

> 💡 Ne corrigez **jamais** un déploiement cassé en modifiant le cluster directement avec
> `kubectl edit` ou en y appliquant un fichier à la main : le prochain `git push` sur `main`
> redéclenchera le pipeline et **écrasera** silencieusement cette correction. Le correctif doit
> toujours passer par un nouveau commit, pas par une intervention manuelle sur le cluster.

---

## 📚 Références

- [Helm — Charts](https://helm.sh/docs/topics/charts/)
- [Helm — Using Helm](https://helm.sh/docs/intro/using_helm/)
- [Helm — helm upgrade](https://helm.sh/docs/helm/helm_upgrade/)
- [Kubernetes — Deployments, Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
- [Azure — Connexion de GitHub Actions à Azure par OIDC](https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect)

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Kubernetes et Helm]({{% relref "/semaine-12/exercice-kubernetes-helm" %}})
pour valider votre compréhension, avant de passer à votre [activité pratique]({{% relref "/semaine-12/activite-pratique" %}}).
{{% /notice %}}
