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

## 3️⃣ Donner à AKS l'accès à votre image GHCR (si elle est privée)

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

  — [Kubernetes — Créer un Secret en fournissant des identifiants en ligne de commande](https://kubernetes.io/fr/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line)

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
