+++
title = "Kubernetes 1/2 : namespace, Deployment, Service"
weight = 2
draft = false
+++

Tout ce qui suit est fondé sur la documentation officielle **Kubernetes** —
[kubernetes.io/docs/concepts](https://kubernetes.io/docs/concepts/) — appliquée à l'application
legacy `petclinic` déjà conteneurisée à la semaine 7 (image publiée sur GHCR, port `9966`,
endpoint `/petclinic/api/vets`).

---

## 1️⃣ D'abord le problème : pourquoi un conteneur seul ne suffit pas

À la semaine 7, vous avez appris à empaqueter `petclinic` dans une image Docker et à la démarrer
avec `docker run -d -p 9966:9966 ...`. Puis, à la page précédente
([Azure Container Instances]({{% relref "/semaine-11/azure-container-instances" %}})), vous l'avez
déployée dans le nuage avec `az container create` — et vous avez vous-même constaté deux
problèmes : son adresse publique peut changer lors d'un redémarrage, et rien ne permet d'en faire
tourner automatiquement plusieurs copies. Généralisons : imaginez que cette application soit un
vrai site, utilisé jour et nuit par de vrais clients :

- **À 3 h du matin, le conteneur plante.** ACI le relance, mais avec une nouvelle adresse — tout
  ce qui pointait vers l'ancienne est cassé, et personne n'a été prévenu.
- **Le trafic est multiplié par dix.** Un seul conteneur n'arrive plus à répondre, et ACI n'a
  aucun mécanisme pour en démarrer d'autres automatiquement.
- **La machine qui héberge le conteneur tombe en panne.** Si tout roulait sur cette seule machine,
  tout disparaît avec elle.

Aucun de ces trois problèmes n'est un problème de code — `petclinic` lui-même est correct. Le
problème, c'est tout ce qui doit entourer une application pour qu'elle survive en continu, sans
personne à côté pour la surveiller. C'est exactement ce que Kubernetes ajoute à un simple moteur de
conteneurs :

> « Kubernetes provides you with a framework to run distributed systems resiliently. It takes
> care of scaling and failover for your application, provides deployment patterns, and more. »
> — [Kubernetes — Overview](https://kubernetes.io/docs/concepts/overview/)

Deux de ces capacités seront centrales dans cette page : la répartition de charge (le travail du
**Service**, section 4️⃣) et l'auto-guérison, c'est-à-dire le redémarrage automatique (le travail du
**Deployment**, section 3️⃣). Mais avant d'y arriver, il faut d'abord comprendre *où*, concrètement,
tout cela se passe : sur un **cluster**.

### Qu'est-ce qu'un cluster, au juste?

Un **cluster** — le mot anglais signifie « grappe » ou « groupe » — est un ensemble de machines
reliées entre elles et gérées comme si c'était une seule grosse machine :

> « A Kubernetes cluster consists of a control plane and one or more worker nodes. »
> — [Kubernetes — Components](https://kubernetes.io/docs/concepts/overview/components/)

Quand une machine fait partie d'un cluster, on ne l'appelle plus « machine », mais un **nœud**
(*node*) — comme un nœud dans un filet, un point du réseau qui tient avec les autres :

> « Kubernetes runs your workload by placing containers into Pods to run on _Nodes_. A node may
> be a virtual or physical machine, depending on the cluster. [...] Typically you have several
> nodes in a cluster; in a learning or resource-limited environment, you might have only one
> node. »
> — [Kubernetes — Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)

> 🍽️ **Analogie** : un conteneur seul, c'est un cuisinier seul dans sa cuisine — tant qu'il est en
> forme et qu'il y a trois clients, tout va bien. Un cluster, c'est la brigade d'un vrai
> restaurant : plusieurs cuisiniers (des **nœuds**) qui se partagent le travail. Si l'un tombe
> malade, les autres continuent le service. Si la salle se remplit, on appelle du renfort. Le
> client, lui, ne voit jamais la cuisine : il voit un seul restaurant, avec une seule porte
> d'entrée. C'est exactement la promesse d'un cluster : plusieurs machines derrière, mais une
> seule chose à gérer devant.

Dans ce cours, le cluster de chaque équipe sera petit — un seul nœud suffit pour apprendre, et il
coûte moins cher sur votre crédit étudiant (voir
[Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}})). Le
principe reste identique à celui des entreprises qui en ont des centaines.

### Mais qui décide quoi tourne où? Le control plane

Avoir plusieurs machines ne règle rien, à elles seules : il faut quelqu'un qui décide quel nœud
fait rouler quel conteneur, qui surveille en permanence, et qui réagit quand quelque chose casse.
Au sein d'un cluster, les nœuds se divisent donc en deux catégories :

> « A Kubernetes cluster consists of a control plane and one or more worker nodes. »
> — [Kubernetes — Components](https://kubernetes.io/docs/concepts/overview/components/)

| Composant | Rôle (traduit de la documentation officielle) |
|---|---|
| **Control plane** | « Manage the overall state of the cluster » — décide *quoi* faire tourner et *où* |
| **kube-apiserver** | « The core component server that exposes the Kubernetes HTTP API » — le seul point d'entrée pour `kubectl` |
| **kube-scheduler** | « Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node » |
| **Nœuds (nodes)** | « Run on every node, maintaining running pods and providing the Kubernetes runtime environment » |
| **kubelet** | « Ensures that Pods are running, including their containers » — l'agent qui, sur chaque nœud, fait réellement démarrer vos conteneurs |
| **kube-proxy** | « Maintains network rules on nodes to implement Services » |

— [Kubernetes — Components](https://kubernetes.io/docs/concepts/overview/components/)

> 🍽️ **Analogie** (suite) : dans le restaurant, le **control plane**, c'est le chef qui tient les
> commandes, répartit les plats entre les cuisiniers et vérifie que rien ne brûle — mais qui ne
> cuisine pas lui-même. Les **nœuds** sont les cuisiniers.

### Déclarer un état désiré, plutôt que donner des ordres un par un

Vous pourriez imaginer piloter un cluster avec des commandes, une par une : « démarre ceci »,
« arrête cela ». Ce n'est **pas** comme ça que Kubernetes fonctionne.

> 🌡️ **Analogie** : un thermostat ne vous demande pas d'allumer et d'éteindre le chauffage chaque
> minute. Vous affichez la température souhaitée — 21 degrés — et il compare sans arrêt la
> température réelle à la cible, puis corrige tout seul. Kubernetes fait la même chose avec vos
> conteneurs : vous affichez « je veux 2 copies de `petclinic` », et il compare sans arrêt la
> réalité à cette cible, puis corrige tout seul.

En résumé : vous décrivez un **état désiré** (« je veux 2 copies de `petclinic` qui tournent »)
dans un fichier YAML envoyé au control plane via `kubectl`, et le control plane se charge, en
continu, de faire correspondre l'état réel du cluster à cet état désiré — y compris en relançant
des conteneurs disparus. On appelle cette surveillance permanente la **réconciliation**, et cette
façon de travailler — décrire le résultat voulu plutôt que les gestes un par un — le **mode
déclaratif**. C'est cette boucle de contrôle continue, et non un script exécuté une seule fois,
qui distingue fondamentalement Kubernetes d'un `docker run` manuel.

> 💡 Dans ce cours, vous n'installerez jamais vous-mêmes un control plane : votre cluster
> [Azure Kubernetes Service]({{% relref "/semaine-11/azure-kubernetes-service" %}}) est déjà prêt.
> Votre travail se limite à y envoyer des manifestes YAML avec `kubectl apply`.

---

## 2️⃣ `namespace` : cloisonner les ressources d'un cluster

Un cluster Kubernetes est souvent **partagé** entre plusieurs équipes ou projets — dans une
entreprise, le même cluster héberge fréquemment plusieurs applications, parfois de plusieurs
équipes. Le mécanisme qui permet ce partage, sans que les ressources des uns et des autres se
mélangent, s'appelle le **namespace** :

> « In Kubernetes, _namespaces_ provide a mechanism for isolating groups of resources within a
> single cluster. Names of resources need to be unique within a namespace, but not across
> namespaces. »
> — [Kubernetes — Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

> « Namespaces are intended for use in environments with many users spread across multiple teams,
> or projects. [...] Namespaces are a way to divide cluster resources between multiple users (via
> resource quota). »
> — [Kubernetes — Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

> 🏢 **Analogie** : les appartements d'un immeuble. L'appartement 301 et le 401 ont tous deux une
> « cuisine », sans confusion, parce que l'adresse complète inclut le numéro d'appartement. Deux
> noms identiques (`petclinic`) peuvent coexister dans le même cluster, à condition de vivre dans
> deux namespaces différents.

Dans ce cours, **chaque équipe possède son propre cluster AKS** (voir
[Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}})) : vous
n'avez donc personne avec qui partager le vôtre, et travailler dans le namespace `default` suffit
amplement. Mais comprendre le mot est indispensable : dès que vous partagerez un cluster en
entreprise (ou avec d'autres équipes du cours, dans un exercice futur), la toute première question
à vous poser sera « dans quel namespace suis-je? ».

Créer un namespace et y diriger vos commandes `kubectl`, si vous souhaitez tout de même pratiquer :

```shell
kubectl create namespace equipe-04
kubectl config set-context --current --namespace=equipe-04
# Validation :
kubectl config view --minify | grep namespace:
```

— [Kubernetes — Namespaces, Setting the namespace preference](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#setting-the-namespace-preference)

> ⚠️ La documentation officielle est explicite : « For a production cluster, consider _not_ using
> the `default` namespace. » ([Kubernetes — Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/))
> En entreprise, sur un cluster partagé, ne déployez jamais dans `default` — sans quoi vos
> ressources se mélangent avec celles des autres équipes ou projets du même cluster.

---

## 3️⃣ `Deployment` : réplicas et auto-guérison

### Une chaîne de problèmes, pas une liste à mémoriser

Ne retenez pas Pod, Deployment et Service comme un vocabulaire isolé : chaque objet naît du défaut
du précédent.

**Étape 1 — le conteneur ne suffit pas, alors on invente le Pod.** Kubernetes ne manipule pas vos
conteneurs Docker directement : il lui faut sa propre unité, qu'il peut créer, déplacer, surveiller
et remplacer. Cette unité, c'est le **Pod** :

> « Pods are the smallest deployable units of computing that you can create and manage in
> Kubernetes. A Pod [...] is a group of one or more containers, with shared storage and network
> resources, and a specification for how to run the containers. [...] The "one-container-per-Pod"
> model is the most common Kubernetes use case; in this case, you can think of a Pod as a
> wrapper around a single container. »
> — [Kubernetes — Pods](https://kubernetes.io/docs/concepts/workloads/pods/)

> 💊 **Analogie** : le conteneur est le médicament, le Pod est la capsule qui permet au corps de
> l'utiliser. Point crucial : un Pod est **jetable**. Il peut mourir, et son remplaçant aura une
> autre adresse IP. Tout le reste de cette page découle de ce seul fait.

**Étape 2 — les Pods meurent, alors on invente le Deployment.** Si les Pods sont jetables, qui
s'assure qu'il y en a toujours assez? Vous n'allez pas en recréer à la main chaque fois. C'est le
rôle du **Deployment**, détaillé ci-dessous. **Étape 3 — les adresses changent, alors on invente le
Service**, détaillé en section 4️⃣.

### Le Deployment

Un **Deployment** décrit, de façon déclarative, l'état désiré d'un ensemble de Pods identiques :

> « You describe a _desired state_ in a Deployment, and the Deployment Controller changes the
> actual state to the desired state at a controlled rate. »
> — [Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

> 🏪 **Analogie** : le Deployment est le gérant d'un magasin qui a décidé qu'il fallait toujours
> 2 caissiers (des **réplicas**, c'est-à-dire des copies) : si l'un part en pause, il en appelle
> un autre, sans que le propriétaire ait à intervenir.

Voici un Deployment complet pour l'application `petclinic` (à adapter avec le nom de votre propre
image GHCR, publiée à la semaine 7) :

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
      containers:
        - name: petclinic
          image: ghcr.io/<votre-compte>/petclinic:latest
          ports:
            - containerPort: 9966
```

| Champ | Rôle |
|---|---|
| `apiVersion: apps/v1` | Groupe et version de l'API Kubernetes qui définit l'objet `Deployment` |
| `metadata.name` | Nom du Deployment — sert de base au nom des Pods créés (ex. `petclinic-75675f5897-7ci7o`) |
| `spec.replicas` | Nombre de copies (Pods) désirées — ici 2, pour absorber la charge et tolérer la perte d'une copie |
| `spec.selector.matchLabels` | Étiquette utilisée par le Deployment pour reconnaître **ses** Pods — doit correspondre à `template.metadata.labels` |
| `spec.template` | Le « moule » utilisé pour créer chaque Pod : ses étiquettes et sa spécification de conteneur |
| `containers[0].image` | Image Docker à exécuter — ici, l'image `petclinic` publiée sur GHCR à la semaine 7 |
| `containers[0].ports.containerPort` | Port écouté par l'application à l'intérieur du conteneur — `9966`, comme dans l'`EXPOSE` du `Dockerfile` (semaine 7) |

Appliquer ce manifeste, puis vérifier son état :

```shell
kubectl apply -f deployment.yaml
kubectl get deployments
# NAME        READY   UP-TO-DATE   AVAILABLE   AGE
# petclinic   2/2     2            2           18s
```

— [Kubernetes — Deployments, Creating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)

C'est ce même contrôleur qui réalise l'**auto-guérison** mentionnée en section 1️⃣ : si un Pod
plante ou est supprimé, le Deployment en recrée immédiatement un nouveau pour revenir à l'état
désiré (`replicas: 2`) — sans qu'aucun humain n'intervienne.

---

## 4️⃣ `Service` : exposer le Deployment et répartir la charge

Le problème que résout un **Service** est décrit ainsi par la documentation officielle : les Pods
créés par un Deployment sont **éphémères** (ils apparaissent et disparaissent), et chacun reçoit
sa propre adresse IP.

> « Pods are ephemeral resources (you should not expect that an individual Pod is reliable and
> durable). [...] This leads to a problem: if some set of Pods ("backends") provides
> functionality to other Pods [...], how do the frontends find out and keep track of which IP
> address to connect to? [...] Enter _Services_. »
> — [Kubernetes — Service](https://kubernetes.io/docs/concepts/services-networking/service/)

> ☎️ **Analogie** : le numéro de téléphone du standard d'une entreprise. Les employés changent de
> bureau et de poste (les Pods sont remplacés, leur IP change), mais le client compose toujours
> le même numéro, et le standard le relie à quelqu'un de disponible. Sans Service, votre
> application serait un commerce dont le numéro changerait chaque matin.

Un Service pour exposer le Deployment `petclinic` défini plus haut :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: petclinic-service
spec:
  type: LoadBalancer
  selector:
    app: petclinic
  ports:
    - protocol: TCP
      port: 9966
      targetPort: 9966
```

| Champ | Rôle |
|---|---|
| `metadata.name` | Nom du Service — devient aussi un nom DNS interne au cluster |
| `spec.type` | Mode d'exposition (voir tableau des types ci-dessous) |
| `spec.selector` | Doit correspondre aux `labels` des Pods à cibler (`app: petclinic`) — c'est ainsi que le Service retrouve **tous** les Pods du Deployment, peu importe leur nombre |
| `ports.port` | Port exposé par le **Service** lui-même |
| `ports.targetPort` | Port réellement écouté par le conteneur (`containerPort` du Deployment) — ici les deux valent `9966`, comme dans le Dockerfile de la semaine 7 |

### `ClusterIP` vs `LoadBalancer`

> « `ClusterIP` Exposes the Service on a cluster-internal IP. Choosing this value makes the
> Service only reachable from within the cluster. This is the default. »
> « `LoadBalancer` Exposes the Service externally using an external load balancer. Kubernetes
> does not directly offer a load balancing component; you must provide one, or you can integrate
> your Kubernetes cluster with a cloud provider. »
> — [Kubernetes — Service, Service type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)

| Type | Accessible depuis | Cas d'usage typique dans ce cours |
|---|---|---|
| `ClusterIP` (défaut) | Uniquement l'intérieur du cluster | Une base de données interne qu'aucun humain n'appelle directement |
| `LoadBalancer` | L'extérieur du cluster, via une IP publique | `petclinic`, pour y accéder depuis votre navigateur |

### Répartition de charge

C'est le même mécanisme de Service qui répartit le trafic entre les Pods du Deployment :
le **selector** du Service cible toutes les Pods portant l'étiquette `app: petclinic` — qu'il y en
ait 2 ou 10 — et le Service continue de distribuer les requêtes entrantes entre elles, exactement
ce que décrit la capacité « Service discovery and load balancing » de la section 1️⃣
([Kubernetes — Overview](https://kubernetes.io/docs/concepts/overview/)).

Appliquer et vérifier :

```shell
kubectl apply -f service.yaml
kubectl get services
curl http://<ip-externe>:9966/petclinic/api/vets
```

---

> 💡 **Comment les pièces travaillent ensemble**
>
> 1. Le **cluster** de votre équipe contient un ou plusieurs **nœuds**.
> 2. Le **Deployment** garde en permanence `replicas: 2` **Pods** de `petclinic` en vie — il en
>    relance un automatiquement si l'un plante (auto-guérison).
> 3. Le **Service** cible ces mêmes Pods via leur étiquette (`app: petclinic`) et répartit le
>    trafic entrant entre elles, sans jamais exposer leurs adresses IP individuelles (éphémères)
>    à l'extérieur.
> 4. Le **namespace** (`default`, dans ce cours) cloisonne tout cela à l'intérieur du cluster.

---

<details>
<summary>🤔 Testez-vous</summary>

**Question** : Votre Deployment `petclinic` a `replicas: 3`. Un des trois Pods plante à cause d'une
erreur mémoire. Que se passe-t-il, en vous appuyant sur ce que vous venez de lire — et le Service
doit-il être modifié pour continuer à fonctionner?

**Réponse** : Le Deployment Controller détecte que l'état réel (2 Pods vivants) ne correspond plus
à l'état désiré (`replicas: 3`) et recrée automatiquement un nouveau Pod pour revenir à 3 — c'est
l'auto-guérison décrite dans la documentation officielle (section 1️⃣). Le Service, lui, n'a besoin
d'**aucune** modification : son `selector` (`app: petclinic`) cible l'étiquette, pas les Pods
individuels par leur IP ou leur nom; dès que le nouveau Pod porte la même étiquette, le Service
l'inclut automatiquement dans la répartition de charge.

</details>

---

## 📚 Références

- Documentation officielle Kubernetes — [Overview](https://kubernetes.io/docs/concepts/overview/).
- Documentation officielle Kubernetes — [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/).
- Documentation officielle Kubernetes — [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/).
- Documentation officielle Kubernetes — [Pods](https://kubernetes.io/docs/concepts/workloads/pods/).
- Documentation officielle Kubernetes — [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).
- Documentation officielle Kubernetes — [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).
- Documentation officielle Kubernetes — [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}})
pour découvrir le cluster managé sur lequel vous allez appliquer ces mêmes manifestes, et comment
gérer votre crédit étudiant Azure.
{{% /notice %}}
