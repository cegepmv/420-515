+++
title = "Azure Container Instances (ACI) : déployer un conteneur, seul"
weight = 1
draft = false
+++

Avant de voir Kubernetes, déployons `petclinic` (semaine 7) de la façon la plus simple possible
dans le nuage : **un seul conteneur, une adresse publique, rien d'autre**. C'est exactement ce que
fait **Azure Container Instances (ACI)** — et c'est en touchant ses limites, à la main, que la
nécessité de Kubernetes deviendra concrète.

> « Azure Container Instances offers the fastest and simplest way to run Linux or Windows
> containers in Azure, without having to manage any virtual machines and without having to adopt
> a higher-level service. [...] Deploy an application to a container instance on-demand when you
> don't need a full container orchestration platform like Azure Kubernetes Service. »
> — [Microsoft Learn — Container instances overview](https://learn.microsoft.com/azure/container-instances/container-instances-overview),
> [Quickstart](https://learn.microsoft.com/azure/container-instances/container-instances-quickstart)

---

## 1️⃣ Créer un groupe de ressources

Comme toute ressource Azure, un conteneur ACI vit dans un **groupe de ressources** — la boîte qui
range tout ce qui appartient au même projet (voir aussi
[Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}}) pour la
même notion appliquée à AKS). Remplacez `equipe04` par votre identifiant d'équipe :

```shell
az login
az account set --subscription "Azure for Students"

az group create --name rg-equipe04-aci --location canadacentral
```

— [Microsoft Learn — Quickstart: Deploy container instance](https://learn.microsoft.com/azure/container-instances/container-instances-quickstart#create-a-resource-group)

---

## 2️⃣ Déployer le conteneur

### Si votre image GHCR (semaine 7) est publique

```shell
az container create \
  --resource-group rg-equipe04-aci \
  --name aci-equipe04 \
  --image ghcr.io/<votre-compte>/petclinic:latest \
  --dns-name-label aci-equipe04 \
  --ports 9966 \
  --os-type Linux \
  --cpu 1 \
  --memory 1.5
```

### Si votre image GHCR est restée privée

Ajoutez les identifiants du registre (même compte GitHub et même PAT `read:packages` que pour
AKS, voir [Azure Kubernetes Service (AKS), section 3️⃣]({{% relref "/semaine-11/azure-kubernetes-service" %}})) :

```shell
az container create \
  --resource-group rg-equipe04-aci \
  --name aci-equipe04 \
  --image ghcr.io/<votre-compte>/petclinic:latest \
  --registry-login-server ghcr.io \
  --registry-username <votre-compte-github> \
  --registry-password <votre-PAT-read:packages> \
  --dns-name-label aci-equipe04 \
  --ports 9966 \
  --os-type Linux \
  --cpu 1 \
  --memory 1.5
```

— [Microsoft Learn — az container create](https://learn.microsoft.com/cli/azure/container?view=azure-cli-latest#az-container-create)

> 💡 `--dns-name-label` doit être **unique** dans toute la région Azure choisie — s'il est déjà
> pris, Azure refuse la création avec l'erreur « DNS name label not available » ; essayez un autre
> nom (ex. `aci-equipe04-v2`).

---

## 3️⃣ Trouver l'adresse publique et tester l'application

```shell
az container show \
  --resource-group rg-equipe04-aci \
  --name aci-equipe04 \
  --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
  --out table
```

> « When you run the command, the container's fully qualified domain name (FQDN) and its
> provisioning state are displayed. [...] If the container's `ProvisioningState` is **Succeeded**,
> go to its FQDN in your browser. »
> — [Microsoft Learn — Quickstart: Deploy container instance](https://learn.microsoft.com/azure/container-instances/container-instances-quickstart#view-the-app)

Résultat attendu : une ligne `Succeeded`, avec un FQDN de la forme
`aci-equipe04.canadacentral.azurecontainer.io`. Testez :

```shell
curl http://aci-equipe04.canadacentral.azurecontainer.io:9966/petclinic/api/vets
```

Si ça ne répond pas tout de suite, attendez quelques secondes (propagation DNS) et réessayez. En
cas de problème, consultez les journaux du conteneur :

```shell
az container logs --resource-group rg-equipe04-aci --name aci-equipe04
```

— [Microsoft Learn — Quickstart, Pull the container logs](https://learn.microsoft.com/azure/container-instances/container-instances-quickstart#pull-the-container-logs)

🎉 **À ce stade, votre application tourne dans le nuage, avec une adresse publique — en une seule
commande `az container create`, sans cluster, sans YAML.** Gardez ce FQDN ouvert dans votre
navigateur pour la suite.

---

## 4️⃣ Tester les limites, à la main

### ACI redémarre un conteneur planté — mais change son adresse

Par défaut, ACI applique la politique `Always` : un conteneur qui plante **est bel et bien
redémarré automatiquement**, sans action de votre part.

> « `Always` Containers in the container group are always restarted. This policy is the
> **default** setting applied when no restart policy is specified at container creation. »
> — [Microsoft Learn — Restart policy for Azure Container Instances](https://learn.microsoft.com/azure/container-instances/container-instances-restart-policy)

Mais la documentation officielle prévient d'un détail crucial :

> « If your container group is configured with an IP address, that IP address can change when
> the container group is restarted. »
> — [Microsoft Learn — Restart policy for Azure Container Instances](https://learn.microsoft.com/azure/container-instances/container-instances-restart-policy)

Forcez un redémarrage pour l'observer :

```shell
az container restart --resource-group rg-equipe04-aci --name aci-equipe04
az container show --resource-group rg-equipe04-aci --name aci-equipe04 \
  --query "ipAddress.ip" --out tsv
```

> ⚠️ Comparez l'adresse IP affichée avant et après le redémarrage. Si elle a changé, tout ce qui
> pointait vers l'ancienne adresse (un autre service, un signet, une configuration DNS manuelle)
> est maintenant cassé — personne n'a été prévenu.

### Pas de deuxième copie pour absorber la charge

Essayez maintenant d'augmenter le nombre de copies de `petclinic`, comme vous le feriez avec
`kubectl scale` en Kubernetes. Il n'existe **aucune commande ACI équivalente** : un groupe de
conteneurs ACI exécute un ensemble fixe de conteneurs, décrit une fois à la création — il n'y a
pas de notion de « réplicas » déclarés qu'Azure maintiendrait pour vous. Pour faire tourner une
deuxième copie, il faudrait créer, à la main, un **deuxième** groupe de conteneurs (`aci-equipe04-2`),
avec sa **propre** adresse — et vous devriez vous-même construire un mécanisme pour répartir le
trafic entre les deux.

---

## 5️⃣ Nettoyer

ACI est facturé par seconde d'exécution — supprimez le groupe de ressources dès que vous avez
terminé cette exploration, pour ne pas consommer votre crédit étudiant inutilement :

```shell
az group delete --name rg-equipe04-aci --yes --no-wait
```

---

> 💡 **Ce que vous venez de découvrir, sans qu'on vous l'ait seulement raconté** :
>
> 1. ACI redémarre un conteneur planté — mais son **adresse peut changer**, cassant silencieusement
>    tout ce qui en dépendait.
> 2. ACI n'offre **aucun mécanisme natif** pour maintenir plusieurs copies identiques d'une
>    application, ni pour répartir le trafic entre elles.
>
> Ce sont précisément les deux problèmes que Kubernetes résout — avec une adresse **stable** qui
> ne bouge jamais (le **Service**) et un nombre de copies **maintenu automatiquement** (le
> **Deployment**). Poursuivez avec
> [Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}}) pour voir
> comment.

---

## 📚 Références

- Microsoft Learn — [Serverless containers in Azure — Container Instances overview](https://learn.microsoft.com/azure/container-instances/container-instances-overview)
- Microsoft Learn — [Quickstart: Deploy a container instance using the Azure CLI](https://learn.microsoft.com/azure/container-instances/container-instances-quickstart)
- Microsoft Learn — [az container create](https://learn.microsoft.com/cli/azure/container?view=azure-cli-latest#az-container-create)
- Microsoft Learn — [Restart policy for Azure Container Instances](https://learn.microsoft.com/azure/container-instances/container-instances-restart-policy)

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}})
pour voir comment namespace, Deployment et Service règlent, l'un après l'autre, les limites que
vous venez de constater vous-même sur ACI.
{{% /notice %}}
