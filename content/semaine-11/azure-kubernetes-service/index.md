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

> « Azure for Students [...] Available only to full-time university students* [...] No credit card
> required [...] Access to full catalog of services up to free amounts and $100 credit. »
> — [Microsoft Azure — Azure for Students](https://azure.microsoft.com/free/students/)

**Avant de pouvoir créer quoi que ce soit sur AKS**, chaque équipe doit créer son propre compte
**Azure for Students**, avec un courriel institutionnel du collège, à partir de la page
[azure.microsoft.com/free/students](https://azure.microsoft.com/free/students/) — c'est ce compte
qui active le crédit de 100 $ utilisé pour provisionner le cluster AKS de l'équipe.

Pourquoi un **abonnement** : quelqu'un doit payer les machines, et Azure doit savoir qui. Un
abonnement (*subscription*) est le compte auquel la facture est rattachée. Chaque équipe a le sien,
avec son crédit Azure for Students — c'est pour cela qu'un cluster ne peut pas être « à tout le
monde » : il appartient à un seul abonnement.

---

## 2️⃣ Choisir ses noms, puis créer le cluster

### Une convention de nommage, à adopter dès le départ

Chaque équipe choisit un identifiant court, unique, en minuscules, sans accent (ex. `equipe04`).
Tous les noms Azure de l'équipe en découlent :

| Ressource | Modèle | Exemple rempli (`equipe04`) |
|---|---|---|
| Groupe de ressources | `rg-<id>` | `rg-equipe04` |
| Cluster AKS | `aks-<id>` | `aks-equipe04` |
| Image GHCR (semaine 7) | `ghcr.io/<compte-github>/petclinic:v1` | `ghcr.io/equipe04/petclinic:v1` |

Pourquoi regrouper les ressources : un projet crée vite un fouillis — un cluster ici, un disque
ailleurs — et à la fin, on ne sait plus quoi supprimer sans risque. Le **groupe de ressources**
(*resource group*) est la boîte étiquetée où l'on range dès le départ tout ce qui appartient au
même projet. Supprimer le groupe supprime tout son contenu : c'est le nettoyage de fin de session,
en une seule commande.

### Créer le groupe de ressources, puis le cluster

La création d'un cluster prend 10 à 15 minutes d'attente — prévoyez ce délai avant votre première
séance de travail plutôt que de l'absorber en classe. Remplacez `equipe04` par votre propre
identifiant dans toutes les commandes suivantes :

```shell
az login
az account set --subscription "Azure for Students"

# 1. Le groupe de ressources qui contiendra tout
az group create --name rg-equipe04 --location canadacentral

# 2. Le cluster : un seul nœud, petit, pour limiter le coût (10-15 min)
az aks create \
  --resource-group rg-equipe04 \
  --name aks-equipe04 \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys

# 3. Récupérer les identifiants d'accès au cluster
az aks get-credentials \
  --resource-group rg-equipe04 \
  --name aks-equipe04

# 4. Preuve que ça fonctionne
kubectl get nodes
```

— [Microsoft Learn — Create an AKS cluster](https://learn.microsoft.com/azure/aks/learn/quick-kubernetes-deploy-cli)

Résultat attendu à l'étape 4 : une ligne affichant votre nœud avec le statut `Ready`. Si la région
`canadacentral` refuse la création (capacité insuffisante), refaites l'étape 1 avec `--location
eastus` et gardez cette même région pour la suite de la session.

> 💡 `az aks get-credentials` ne fait qu'une chose : il écrit, dans votre fichier `~/.kube/config`,
> l'adresse de votre cluster et votre clé d'accès. Ensuite, c'est `kubectl` qui parle directement
> à votre cluster — ne transmettez jamais ce fichier à quelqu'un d'autre : c'est la clé de votre
> cluster, et de la facture qui vient avec.

---

## 3️⃣ Donner à AKS l'accès à votre image GHCR (si elle est privée)

Par défaut, un *package* publié sur **GHCR** (GitHub Container Registry) est **privé** — personne
d'autre que vous ne peut le télécharger, pas même votre propre cluster AKS. Si votre image
`petclinic` (publiée à la semaine 7) est restée privée, `kubectl apply` créera bien le Deployment,
mais chaque Pod restera bloqué en `ImagePullBackOff` : AKS refuse de tirer une image à laquelle il
n'a pas accès.

> « A Kubernetes cluster uses the Secret of `kubernetes.io/dockerconfigjson` type to authenticate
> with a container registry to pull a private image. »
> — [Kubernetes — Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

Deux façons de régler le problème — choisissez-en **une seule** :

- **Option A — rendre le package public** (le plus simple pour ce cours) : dans GitHub, allez dans
  les paramètres du *package* `petclinic` (lié à votre dépôt, semaine 7) et changez sa visibilité
  à *Public*. Aucune commande Kubernetes n'est alors nécessaire.
- **Option B — garder le package privé**, et donner explicitement à AKS la permission de le tirer,
  avec un [jeton d'accès personnel (PAT)](https://docs.github.com/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
  limité au droit `read:packages` :

  ```shell
  kubectl create secret docker-registry ghcr-secret \
    --docker-server=ghcr.io \
    --docker-username=<votre-compte-github> \
    --docker-password=<votre-PAT-read:packages>
  ```

  — [Kubernetes — Create a Secret by providing credentials on the command line](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line)

  Puis référencez ce secret dans votre Deployment, au même niveau que `containers` :

  ```yaml
  spec:
    imagePullSecrets:
      - name: ghcr-secret
    containers:
      - name: petclinic
        image: ghcr.io/<votre-compte>/petclinic:latest
  ```

---

## 4️⃣ Gérer le crédit : arrêter le cluster quand vous ne travaillez pas

> ⚠️ Un cluster AKS **actif** est facturé en continu, à l'heure, tant qu'il tourne — même si
> personne de l'équipe ne l'utilise activement. Avec seulement 100 $ de crédit pour toute la
> session, laisser un cluster tourner en continu (le soir, la nuit, entre deux séances de travail)
> peut épuiser tout le crédit de l'équipe bien avant la fin de la session.

La documentation officielle Azure recommande précisément d'arrêter un cluster AKS inutilisé pour
réduire ces coûts :

> « You may not need to continuously run your Azure Kubernetes Service (AKS) workloads. [...] To
> better optimize your costs during these periods, you can turn off, or stop, your cluster. This
> action stops your control plane and agent nodes, allowing you to save on all the compute costs,
> while maintaining all objects except standalone pods. The cluster state is stored for when you
> start it again, allowing you to pick up where you left off. »
> — [Microsoft Learn — Stop and start an Azure Kubernetes Service (AKS) cluster](https://learn.microsoft.com/azure/aks/start-stop-cluster)

Pour arrêter le cluster, à la toute fin de **chaque** session de travail (pas seulement le
vendredi) :

```shell
az aks stop --name aks-equipe04 --resource-group rg-equipe04
```

> « Use the `az aks stop` command to stop a running AKS cluster, including the nodes and control
> plane. »
> — [Microsoft Learn — Stop and start an Azure Kubernetes Service (AKS) cluster](https://learn.microsoft.com/azure/aks/start-stop-cluster)

Et pour le redémarrer, au début de votre prochaine séance de travail :

```shell
az aks start --name aks-equipe04 --resource-group rg-equipe04
```

> « Use the `az aks start` command to start a stopped AKS cluster. The cluster restarts with the
> previous control plane state and number of agent nodes. »
> — [Microsoft Learn — Stop and start an Azure Kubernetes Service (AKS) cluster](https://learn.microsoft.com/azure/aks/start-stop-cluster)

> 💡 Après un `az aks stop`, la documentation officielle précise qu'il faut attendre « 15-30
> minutes » avant de redémarrer le cluster, le temps que l'arrêt se termine complètement —
> relancer trop vite peut perturber l'arrêt en cours
> ([Microsoft Learn — Stop and start an AKS cluster](https://learn.microsoft.com/azure/aks/start-stop-cluster)).
> Prenez l'habitude d'arrêter votre cluster **dès que vous fermez votre session de travail**, même
> pour une pause de quelques heures : c'est le seul moyen fiable de ne pas brûler votre crédit
> étudiant avant la fin de la session.

En fin de session (toute la session, pas seulement une séance), supprimez le groupe de ressources :
tout ce qu'il contient — cluster inclus — disparaît en une seule commande, et la facturation
s'arrête complètement.

```shell
az group delete --name rg-equipe04 --yes --no-wait
```

---

## 📚 Références

- Microsoft Azure — [Azure for Students](https://azure.microsoft.com/free/students/)
- Microsoft Learn — [Create an Azure Kubernetes Service (AKS) cluster](https://learn.microsoft.com/azure/aks/learn/quick-kubernetes-deploy-cli)
- Documentation officielle Kubernetes — [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- Documentation officielle Microsoft — [Stop and start an Azure Kubernetes Service (AKS) cluster](https://learn.microsoft.com/azure/aks/start-stop-cluster)

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Kubernetes]({{% relref "/semaine-11/exercice-kubernetes" %}}) pour
valider votre compréhension des namespaces, Deployments, Services, de l'accès à une image GHCR
privée, et de la gestion de votre crédit Azure.
{{% /notice %}}
