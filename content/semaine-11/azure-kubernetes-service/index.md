+++
title = "Azure Kubernetes Service (AKS) : votre cluster dans le nuage"
weight = 3
draft = false
+++

Cette page complète [Kubernetes 1/2 : namespace, Deployment, Service]({{% relref "/semaine-11/kubernetes-fondamentaux" %}})
en expliquant **où**, concrètement, vous allez appliquer ces manifestes : un cluster managé par
Microsoft Azure, financé par votre crédit étudiant.

---

## 1️⃣ Créer son compte et son abonnement Azure

Ce cluster AKS est payé à même le **crédit étudiant Azure**, offert gratuitement aux étudiants à
temps plein :

> « Azure for Students [...] Disponible uniquement pour les étudiants universitaires à plein
> temps* [...] Aucune carte de crédit requise [...] Accès au catalogue complet de services
> jusqu'à des montants gratuits et un crédit de 100 USD. »
> — [Microsoft Azure — Azure for Students](https://azure.microsoft.com/fr-ca/free/students/)

**Avant de pouvoir créer quoi que ce soit sur AKS**, chaque étudiant doit créer son propre compte
**Azure for Students**, avec un courriel institutionnel du collège, à partir de la page
[azure.microsoft.com/fr-ca/free/students](https://azure.microsoft.com/fr-ca/free/students/) —
c'est ce compte qui active le crédit de 100 $ utilisé pour provisionner **votre propre** cluster
AKS, personnel.

Pourquoi un **abonnement** : quelqu'un doit payer les machines, et Azure doit savoir qui. Un
abonnement (*subscription*) est le compte auquel la facture est rattachée. Chaque étudiant a le
sien, avec son propre crédit Azure for Students — c'est pour cela qu'un cluster ne peut pas être
« à tout le monde » : il appartient à un seul abonnement, celui de la personne qui l'a créé.

---

## 2️⃣ Choisir ses noms, puis créer le cluster

### Une convention de nommage, à adopter dès le départ

Choisissez un identifiant court, unique, en minuscules, sans accent, basé sur votre matricule ou
votre nom d'utilisateur GitHub (ex. `etu1234`). Tous vos noms Azure en découlent :

| Ressource | Modèle | Exemple rempli (`etu1234`) |
|---|---|---|
| Groupe de ressources | `rg-<id>` | `rg-etu1234` |
| Cluster AKS | `aks-<id>` | `aks-etu1234` |
| Image GHCR (semaine 7) | `ghcr.io/<compte-github>/petclinic:v1` | `ghcr.io/etu1234/petclinic:v1` |

Pourquoi regrouper les ressources : un projet crée vite un fouillis — un cluster ici, un disque
ailleurs — et à la fin, on ne sait plus quoi supprimer sans risque. Le **groupe de ressources**
(*resource group*) est la boîte étiquetée où l'on range dès le départ tout ce qui appartient au
même projet. Supprimer le groupe supprime tout son contenu : c'est le nettoyage de fin de session,
en une seule commande.

### Créer le groupe de ressources, puis le cluster

La création d'un cluster prend 10 à 15 minutes d'attente — prévoyez ce délai avant votre première
séance de travail plutôt que de l'absorber en classe. Remplacez `etu1234` par votre propre
identifiant dans toutes les commandes suivantes :

```shell
az login
az account set --subscription "Azure for Students"

# 1. Le groupe de ressources qui contiendra tout
az group create --name rg-etu1234 --location canadacentral

# 2. Le cluster : un seul nœud, petit, pour limiter le coût (10-15 min)
az aks create \
  --resource-group rg-etu1234 \
  --name aks-etu1234 \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys

# 3. Récupérer les identifiants d'accès au cluster
az aks get-credentials \
  --resource-group rg-etu1234 \
  --name aks-etu1234

# 4. Preuve que ça fonctionne
kubectl get nodes
```

— [Microsoft Learn — Créer un cluster AKS](https://learn.microsoft.com/fr-ca/azure/aks/learn/quick-kubernetes-deploy-cli)

Résultat attendu à l'étape 4 : une ligne affichant votre nœud avec le statut `Ready`. Si la région
`canadacentral` refuse la création (capacité insuffisante), refaites l'étape 1 avec `--location
eastus` et gardez cette même région pour la suite de la session.

> 💡 `az aks get-credentials` ne fait qu'une chose : il écrit, dans votre fichier `~/.kube/config`,
> l'adresse de votre cluster et votre clé d'accès. Ensuite, c'est `kubectl` qui parle directement
> à votre cluster — ne transmettez jamais ce fichier à quelqu'un d'autre : c'est la clé de votre
> cluster, et de la facture qui vient avec.

---

## 3️⃣ Créer son namespace et s'assurer d'y travailler

`az aks get-credentials` (étape précédente) configure `kubectl` pour parler à votre cluster, mais
vous arrivez par défaut dans le namespace `default` — voir
[Kubernetes 1/2 : namespace, Deployment, Service]({{% relref "/semaine-11/kubernetes-fondamentaux" %}})
pour une explication complète du concept. Même si, dans ce cours, un cluster `default` suffirait
techniquement (votre cluster vous appartient entièrement), **créez et utilisez systématiquement
votre propre namespace** : c'est le réflexe professionnel à prendre dès maintenant, celui que vous
devrez impérativement avoir sur un cluster d'entreprise partagé entre plusieurs équipes.

```shell
# 1. Créer le namespace (remplacez etu1234 par votre identifiant, voir section 2️⃣)
kubectl create namespace etu1234

# 2. Basculer le contexte courant vers ce namespace, pour NE PLUS JAMAIS
#    déployer accidentellement dans "default"
kubectl config set-context --current --namespace=etu1234

# 3. Preuve que vous n'êtes plus dans "default"
kubectl config view --minify | grep namespace:
# namespace: etu1234
```

— [Kubernetes — Namespaces, Définir la préférence de namespace](https://kubernetes.io/fr/docs/concepts/overview/working-with-objects/namespaces/#setting-the-namespace-preference)

> ⚠️ Toutes les commandes `kubectl` qui suivent (secret, Deployment, Service) s'appliquent au
> namespace **courant** de votre contexte — celui que vous venez de fixer à l'étape 2. Si vous
> ouvrez un nouveau terminal plus tard et que `kubectl get pods` revient vide alors que vous êtes
> certain d'avoir déployé votre application, vérifiez d'abord votre contexte avec la commande de
> l'étape 3 : vous êtes probablement repassé dans `default` sans vous en rendre compte (par
> exemple après un nouveau `az aks get-credentials`).

---

## 4️⃣ Donner à AKS l'accès à votre image GHCR (si elle est privée)

Par défaut, un *package* publié sur **GHCR** (GitHub Container Registry) est **privé** — personne
d'autre que vous ne peut le télécharger, pas même votre propre cluster AKS. Si votre image
`petclinic` (publiée à la semaine 7) est restée privée, `kubectl apply` créera bien le Deployment,
mais chaque Pod restera bloqué en `ImagePullBackOff` : AKS refuse de tirer une image à laquelle il
n'a pas accès.

> « Un cluster Kubernetes utilise un Secret de type `kubernetes.io/dockerconfigjson` pour
> s'authentifier auprès d'un registre de conteneurs afin de tirer une image privée. »
> — [Kubernetes — Tirer une image depuis un registre privé](https://kubernetes.io/fr/docs/tasks/configure-pod-container/pull-image-private-registry/)

Deux façons de régler le problème — choisissez-en **une seule** :

- **Option A — rendre le package public** (le plus simple pour ce cours) : dans GitHub, allez dans
  les paramètres du *package* `petclinic` (lié à votre dépôt, semaine 7) et changez sa visibilité
  à *Public*. Aucune commande Kubernetes n'est alors nécessaire — passez directement à la
  section 5️⃣.
- **Option B — garder le package privé**, et donner explicitement à AKS la permission de le tirer,
  avec un [jeton d'accès personnel (PAT)](https://docs.github.com/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
  limité au droit `read:packages`. Le Secret doit exister **dans le même namespace** que votre
  Deployment (`etu1234`, fixé comme contexte courant à la section 3️⃣) — sans `--namespace`
  explicite, il est créé dans le namespace courant du contexte, donc déjà le bon :

  ```shell
  kubectl create secret docker-registry ghcr-secret \
    --namespace=etu1234 \
    --docker-server=ghcr.io \
    --docker-username=<votre-compte-github> \
    --docker-password=<votre-PAT-read:packages>

  # Validation : le secret doit apparaître dans VOTRE namespace
  kubectl get secrets --namespace=etu1234
  ```

  — [Kubernetes — Créer un Secret en fournissant des identifiants en ligne de commande](https://kubernetes.io/fr/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line)

  Créer le Secret ne suffit pas : rien ne dit encore à Kubernetes de **l'utiliser** pour tirer
  l'image de `petclinic`. C'est le rôle du champ `imagePullSecrets`, à ajouter dans votre
  `deployment.yaml`, **avant** de l'appliquer (section 5️⃣) :

  > « Vous pouvez créer un Pod qui utilise un Secret `kubernetes.io/dockerconfigjson` [...] en
  > ajoutant une section `imagePullSecrets` à la définition du Pod. »
  > — [Kubernetes — Tirer une image depuis un registre privé](https://kubernetes.io/fr/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret)

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: petclinic
    labels:
      app: petclinic
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: petclinic
    template:
      metadata:
        labels:
          app: petclinic
      spec:
        imagePullSecrets:
          - name: ghcr-secret
        containers:
          - name: petclinic
            image: ghcr.io/<votre-compte>/petclinic:latest
            ports:
              - containerPort: 9966
  ```

  | Champ | Rôle |
  |---|---|
  | `spec.template.spec.imagePullSecrets` | Liste des Secrets que **kubelet** doit présenter au registre pour tirer l'image — ici, le Secret `ghcr-secret` créé ci-dessus |
  | `imagePullSecrets[0].name` | Doit correspondre **exactement** au nom donné à `kubectl create secret` (`ghcr-secret`) et exister dans le **même namespace** que le Deployment (`etu1234`, section 3️⃣) |

  > ⚠️ `imagePullSecrets` se place au niveau de `spec.template.spec` (la spécification du **Pod**,
  > à l'intérieur du Deployment), **au même niveau que `containers`** — pas à côté de
  > `spec.replicas` ou de `spec.selector`, qui appartiennent au Deployment lui-même. C'est l'erreur
  > la plus fréquente : un `imagePullSecrets` mal indenté est silencieusement ignoré par Kubernetes
  > (`kubectl apply` ne signale aucune erreur), et les Pods restent en `ImagePullBackOff` comme si
  > le Secret n'existait pas.

  Si vous avez choisi l'**Option A** (package public), n'ajoutez **pas** `imagePullSecrets` : ce
  champ est inutile — et sans effet néfaste — pour une image publique, mais il est inutile de le
  laisser traîner dans votre manifeste s'il ne sert à rien.

---

## 5️⃣ Déployer l'application : Deployment, Service, puis trouver l'adresse externe

Avec le namespace prêt (section 3️⃣) et, au besoin, le Secret GHCR en place (section 4️⃣), appliquez
dans l'ordre les deux manifestes décrits et expliqués en détail sur la page
[Kubernetes 1/2 : namespace, Deployment, Service]({{% relref "/semaine-11/kubernetes-fondamentaux" %}}) :

```shell
# 1. Le Deployment : crée les Pods et les maintient en vie (auto-guérison)
kubectl apply -f deployment.yaml

# 2. Vérifier que les réplicas désirés sont bien disponibles avant de continuer
kubectl get deployments
# NAME        READY   UP-TO-DATE   AVAILABLE   AGE
# petclinic   2/2     2            2           18s

# 3. Le Service : expose le Deployment, en LoadBalancer pour une IP publique
kubectl apply -f service.yaml

# 4. Trouver l'adresse externe : la colonne EXTERNAL-IP démarre à <pending>,
#    puis Azure lui attribue une IP publique après quelques dizaines de secondes.
#    --watch rafraîchit l'affichage en continu, sans avoir à relancer la commande.
kubectl get service petclinic-service --watch
# NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
# petclinic-service    LoadBalancer   10.0.123.45    <pending>       9966:31xxx/TCP   5s
# petclinic-service    LoadBalancer   10.0.123.45    20.123.45.67    9966:31xxx/TCP   47s

# 5. Ctrl+C pour quitter le mode --watch une fois l'IP obtenue, puis tester
curl http://20.123.45.67:9966/petclinic/api/vets
```

> 💡 Toutes ces commandes agissent sur le namespace courant de votre contexte (`etu1234`, fixé à
> la section 3️⃣) : pas besoin de répéter `--namespace` à chaque fois. Si vous préférez être
> explicite (ou si vous travaillez depuis un terminal dont vous n'êtes pas sûr du contexte),
> ajoutez `--namespace=etu1234` à chacune de ces commandes.

> ⚠️ Si `kubectl get service` reste bloqué sur `<pending>` plus de quelques minutes, ou si les
> Pods du Deployment restent en `ImagePullBackOff` (`kubectl get pods`), retournez à la
> section 4️⃣ : c'est généralement un problème d'accès à l'image GHCR, pas un problème de Service.

---

## 6️⃣ Gérer le crédit : arrêter le cluster quand vous ne travaillez pas

> ⚠️ Un cluster AKS **actif** est facturé en continu, à l'heure, tant qu'il tourne — même si
> personne ne l'utilise activement. Avec seulement 100 $ de crédit pour toute la session, laisser
> un cluster tourner en continu (le soir, la nuit, entre deux séances de travail) peut épuiser
> tout votre crédit bien avant la fin de la session.

La documentation officielle Azure recommande précisément d'arrêter un cluster AKS inutilisé pour
réduire ces coûts :

> « Il se peut que vous n'ayez pas besoin d'exécuter en continu vos charges de travail Azure
> Kubernetes Service (AKS). [...] Pour mieux optimiser vos coûts pendant ces périodes, vous pouvez
> désactiver, ou arrêter, votre cluster. Cette action arrête votre plan de contrôle et les nœuds de
> l'agent, ce qui vous permet d'économiser sur tous les coûts de calcul, tout en conservant
> l'ensemble de vos objets, à l'exception des pods autonomes. L'état du cluster est conservé pour
> que vous puissiez le redémarrer plus tard et reprendre là où vous vous étiez arrêté. »
> — [Microsoft Learn — Arrêter et démarrer un cluster Azure Kubernetes Service (AKS)](https://learn.microsoft.com/fr-ca/azure/aks/start-stop-cluster)

Pour arrêter le cluster, à la toute fin de **chaque** session de travail (pas seulement le
vendredi) :

```shell
az aks stop --name aks-etu1234 --resource-group rg-etu1234
```

> « Utilisez la commande `az aks stop` pour arrêter un cluster AKS en cours d'exécution, y compris
> les nœuds et le plan de contrôle. »
> — [Microsoft Learn — Arrêter et démarrer un cluster Azure Kubernetes Service (AKS)](https://learn.microsoft.com/fr-ca/azure/aks/start-stop-cluster)

Et pour le redémarrer, au début de votre prochaine séance de travail :

```shell
az aks start --name aks-etu1234 --resource-group rg-etu1234
```

> « Utilisez la commande `az aks start` pour démarrer un cluster AKS arrêté. Le cluster redémarre
> avec l'état précédent de son plan de contrôle et le même nombre de nœuds d'agent. »
> — [Microsoft Learn — Arrêter et démarrer un cluster Azure Kubernetes Service (AKS)](https://learn.microsoft.com/fr-ca/azure/aks/start-stop-cluster)

> 💡 Après un `az aks stop`, la documentation officielle précise qu'il faut attendre « 15 à 30
> minutes » avant de redémarrer le cluster, le temps que l'arrêt se termine complètement —
> relancer trop vite peut perturber l'arrêt en cours
> ([Microsoft Learn — Arrêter et démarrer un cluster AKS](https://learn.microsoft.com/fr-ca/azure/aks/start-stop-cluster)).
> Prenez l'habitude d'arrêter votre cluster **dès que vous fermez votre session de travail**, même
> pour une pause de quelques heures : c'est le seul moyen fiable de ne pas brûler votre crédit
> étudiant avant la fin de la session.

En fin de session (toute la session, pas seulement une séance), supprimez le groupe de ressources :
tout ce qu'il contient — cluster inclus — disparaît en une seule commande, et la facturation
s'arrête complètement.

```shell
az group delete --name rg-etu1234 --yes --no-wait
```

---

## 📚 Références

- Microsoft Azure — [Azure for Students](https://azure.microsoft.com/fr-ca/free/students/)
- Microsoft Learn — [Créer un cluster Azure Kubernetes Service (AKS)](https://learn.microsoft.com/fr-ca/azure/aks/learn/quick-kubernetes-deploy-cli)
- Documentation officielle Kubernetes — [Tirer une image depuis un registre privé](https://kubernetes.io/fr/docs/tasks/configure-pod-container/pull-image-private-registry/)
- Documentation officielle Microsoft — [Arrêter et démarrer un cluster Azure Kubernetes Service (AKS)](https://learn.microsoft.com/fr-ca/azure/aks/start-stop-cluster)

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Kubernetes]({{% relref "/semaine-11/exercice-kubernetes" %}}) pour
valider votre compréhension des namespaces, Deployments, Services, de l'accès à une image GHCR
privée, et de la gestion de votre crédit Azure.
{{% /notice %}}
