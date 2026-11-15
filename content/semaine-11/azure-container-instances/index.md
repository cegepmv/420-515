+++
title = "Azure Container Instances (ACI) : déployer un conteneur, seul"
weight = 1
draft = false
+++

Avant de voir Kubernetes, déployons `petclinic` (semaine 7) de la façon la plus simple possible
dans le nuage : **un seul conteneur, une adresse publique, rien d'autre**. C'est exactement ce que
fait **Azure Container Instances (ACI)** — et c'est en touchant ses limites, à la main, que la
nécessité de Kubernetes deviendra concrète.

> « Azure Container Instances offre le moyen le plus rapide et le plus simple d'exécuter des
> conteneurs Linux ou Windows dans Azure, sans avoir à gérer de machines virtuelles ni à adopter
> un service de niveau supérieur. [...] Déployez une application sur une instance de conteneur à
> la demande quand vous n'avez pas besoin d'une plateforme d'orchestration de conteneur complète
> comme Azure Kubernetes Service. »
> — [Microsoft Learn — Vue d'ensemble d'Azure Container Instances](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-overview),
> [Démarrage rapide](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-quickstart)

---

## 1️⃣ Créer un groupe de ressources

Comme toute ressource Azure, un conteneur ACI vit dans un **groupe de ressources** — la boîte qui
range tout ce qui appartient au même projet (voir aussi
[Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}}) pour la
même notion appliquée à AKS). Remplacez `etu1234` par votre propre identifiant :

```shell
az login
az account set --subscription "Azure for Students"

az group create --name rg-etu1234-aci --location canadacentral
```

— [Microsoft Learn — Démarrage rapide : déployer une instance de conteneur](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-quickstart#create-a-resource-group)

> 💡 **Si c'est la première fois que vous utilisez ACI sur cet abonnement**, la commande
> `az container create` de la section suivante peut échouer avec l'erreur
> `MissingSubscriptionRegistration` pour `Microsoft.ContainerInstance`. C'est normal : ce
> fournisseur de ressources doit d'abord être activé une seule fois sur votre abonnement.
>
> ```shell
> az provider register --namespace Microsoft.ContainerInstance
> # Vérifier que l'enregistrement est terminé (peut prendre 1-2 minutes) :
> az provider show --namespace Microsoft.ContainerInstance --query "registrationState"
> ```
>
> Attendez que la commande affiche `"Registered"`, puis relancez `az container create`.

---

## 2️⃣ Déployer le conteneur

### Si votre image GHCR (semaine 7) est publique

```shell
az container create \
  --resource-group rg-etu1234-aci \
  --name aci-etu1234 \
  --image ghcr.io/<votre-compte>/petclinic:latest \
  --dns-name-label aci-etu1234 \
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
  --resource-group rg-etu1234-aci \
  --name aci-etu1234 \
  --image ghcr.io/<votre-compte>/petclinic:latest \
  --registry-login-server ghcr.io \
  --registry-username <votre-compte-github> \
  --registry-password <votre-PAT-read:packages> \
  --dns-name-label aci-etu1234 \
  --ports 9966 \
  --os-type Linux \
  --cpu 1 \
  --memory 1.5
```

— [Microsoft Learn — az container create](https://learn.microsoft.com/fr-ca/cli/azure/container?view=azure-cli-latest#az-container-create)

> 💡 `--dns-name-label` doit être **unique** dans toute la région Azure choisie — s'il est déjà
> pris, Azure refuse la création avec l'erreur « DNS name label not available » ; essayez un autre
> nom (ex. `aci-etu1234-v2`).

---

## 3️⃣ Trouver l'adresse publique et tester l'application

```shell
az container show \
  --resource-group rg-etu1234-aci \
  --name aci-etu1234 \
  --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
  --out table
```

> « Lorsque vous exécutez la commande, le nom de domaine complet (FQDN) du conteneur et son état
> d'approvisionnement s'affichent. [...] Si l'état `ProvisioningState` du conteneur est
> **Succeeded**, accédez à son FQDN dans votre navigateur. »
> — [Microsoft Learn — Démarrage rapide, afficher l'application](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-quickstart#view-the-app)

Résultat attendu : une ligne `Succeeded`, avec un FQDN de la forme
`aci-etu1234.canadacentral.azurecontainer.io`. Testez :

```shell
curl http://aci-etu1234.canadacentral.azurecontainer.io:9966/petclinic/api/vets
```

Si ça ne répond pas tout de suite, attendez quelques secondes (propagation DNS) et réessayez. En
cas de problème, consultez les journaux du conteneur :

```shell
az container logs --resource-group rg-etu1234-aci --name aci-etu1234
```

— [Microsoft Learn — Démarrage rapide, récupérer les journaux du conteneur](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-quickstart#pull-the-container-logs)

🎉 **À ce stade, votre application tourne dans le nuage, avec une adresse publique — en une seule
commande `az container create`, sans cluster, sans YAML.** Gardez ce FQDN ouvert dans votre
navigateur pour la suite.

---

## 4️⃣ Tester les limites, à la main

### ACI redémarre un conteneur planté — mais change son adresse

Par défaut, ACI applique la politique `Always` : un conteneur qui plante **est bel et bien
redémarré automatiquement**, sans action de votre part.

> « `Always` : les conteneurs du groupe de conteneurs sont toujours redémarrés. Cette stratégie
> est le paramètre **par défaut** appliqué lorsqu'aucune stratégie de redémarrage n'est spécifiée
> à la création du conteneur. »
> — [Microsoft Learn — Stratégie de redémarrage pour Azure Container Instances](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-restart-policy)

Mais la documentation officielle prévient d'un détail crucial :

> « Si votre groupe de conteneurs est configuré avec une adresse IP, cette adresse IP peut changer
> lorsque le groupe de conteneurs est redémarré. »
> — [Microsoft Learn — Stratégie de redémarrage pour Azure Container Instances](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-restart-policy)

Pour l'observer vous-même, notez d'abord l'adresse IP **actuelle**, puis forcez un redémarrage et
comparez :

```shell
# 1. Adresse IP avant le redémarrage
az container show --resource-group rg-etu1234-aci --name aci-etu1234 \
  --query "ipAddress.ip" --out tsv

# 2. Forcer le redémarrage (simule le comportement d'ACI après un plantage)
az container restart --resource-group rg-etu1234-aci --name aci-etu1234

# 3. Attendre que le conteneur soit reparti (quelques dizaines de secondes), puis comparer
az container show --resource-group rg-etu1234-aci --name aci-etu1234 \
  --query "{Etat:provisioningState,IP:ipAddress.ip}" --out table
```

> ⚠️ Comparez l'adresse IP de l'étape 1 et celle de l'étape 3. La documentation officielle dit que
> l'adresse **peut** changer — pas qu'elle changera systématiquement. Avec `az container restart`,
> elle reste d'ailleurs souvent identique en pratique, parce que le même groupe de conteneurs est
> simplement redémarré sur l'infrastructure déjà réservée. Le risque se matérialise surtout dans
> deux cas : un incident Azure qui force une réallocation du conteneur, ou une suppression suivie
> d'une recréation (`az container delete` puis `az container create`) — dans ce dernier cas, l'IP
> change presque toujours, puisqu'une toute nouvelle ressource est provisionnée. Si vous voulez
> provoquer le changement vous-même, essayez :
>
> ```shell
> az container delete --resource-group rg-etu1234-aci --name aci-etu1234 --yes
> # Puis recréez le conteneur (section 2️⃣) et comparez son IP à celle notée à l'étape 1
> ```
>
> Dans les deux cas, le principe reste le même : rien ne garantit la **stabilité** de l'adresse
> IP d'un conteneur ACI dans le temps — c'est l'absence de ce genre de garantie que les Services
> Kubernetes viennent justement résoudre (section suivante). Le FQDN
> (`<label>.<région>.azurecontainer.io`), lui, reste stable tant que le `--dns-name-label` ne
> change pas — c'est l'adresse IP brute qui n'est jamais garantie.

### Pas de deuxième copie pour absorber la charge

Essayez maintenant d'augmenter le nombre de copies de `petclinic`, comme vous le feriez avec
`kubectl scale` en Kubernetes. Il n'existe **aucune commande ACI équivalente** : un groupe de
conteneurs ACI exécute un ensemble fixe de conteneurs, décrit une fois à la création — il n'y a
pas de notion de « réplicas » déclarés qu'Azure maintiendrait pour vous. Pour faire tourner une
deuxième copie, il faudrait créer, à la main, un **deuxième** groupe de conteneurs (`aci-etu1234-2`),
avec sa **propre** adresse — et vous devriez vous-même construire un mécanisme pour répartir le
trafic entre les deux.

---

## 5️⃣ Nettoyer

ACI est facturé par seconde d'exécution — supprimez le groupe de ressources dès que vous avez
terminé cette exploration, pour ne pas consommer votre crédit étudiant inutilement :

```shell
az group delete --name rg-etu1234-aci --yes --no-wait
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

- Microsoft Learn — [Conteneurs serverless dans Azure — Vue d'ensemble de Container Instances](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-overview)
- Microsoft Learn — [Démarrage rapide : déployer une instance de conteneur avec Azure CLI](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-quickstart)
- Microsoft Learn — [az container create](https://learn.microsoft.com/fr-ca/cli/azure/container?view=azure-cli-latest#az-container-create)
- Microsoft Learn — [Stratégie de redémarrage pour Azure Container Instances](https://learn.microsoft.com/fr-ca/azure/container-instances/container-instances-restart-policy)

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}})
pour voir comment namespace, Deployment et Service règlent, l'un après l'autre, les limites que
vous venez de constater vous-même sur ACI.
{{% /notice %}}
