+++
title = "Configuration, secrets et exposition : ConfigMap, Secret, logs, Ingress"
weight = 1
draft = false
+++

Tout ce qui suit est fondé sur la documentation officielle de Kubernetes
(`kubernetes.io`). Les exemples YAML réutilisent le Deployment/Service `petclinic` introduit à la
semaine 11 ([Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}})).

> 💡 **Rappel** : comme à la semaine 11, tout ce qui suit s'applique dans le namespace
> **courant de votre contexte** (`etu1234`, fixé avec
> `kubectl config set-context --current --namespace=...`, voir
> [Azure Kubernetes Service (AKS), section 3️⃣]({{% relref "/semaine-11/azure-kubernetes-service" %}})).
> Si une commande `kubectl get` semble vide, vérifiez d'abord ce contexte
> (`kubectl config view --minify | grep namespace:`) avant de chercher ailleurs.

---

## 1️⃣ Externaliser la configuration : ConfigMap

### Le même problème qu'en Docker, posé une couche plus haut

À la semaine 7, le Dockerfile de `petclinic` évitait déjà de coder en dur des valeurs
spécifiques à un environnement — un port par défaut remplaçable au `docker run`, par exemple. Un
Deployment Kubernetes pose exactement le même problème, à une échelle plus grande : l'URL de la
base de données en développement n'est pas celle de la production, mais l'**image** Docker, elle,
doit rester strictement identique entre les deux — sans quoi vous perdez la garantie, déjà vue à
la semaine 7, qu'un conteneur testé est le conteneur réellement déployé. Un **ConfigMap** règle ce
problème pour toute donnée **non confidentielle** :

> « Une ConfigMap est un objet API utilisé pour stocker des données non confidentielles sous forme
> de paires clé-valeur. [...] Une ConfigMap vous permet de découpler la configuration spécifique à
> un environnement de vos images de conteneur, afin que vos applications soient facilement
> portables. »
> — [kubernetes.io — ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

La documentation précise aussi explicitement la limite d'un ConfigMap :

> « Attention : une ConfigMap ne fournit ni confidentialité ni chiffrement. Si les données que vous
> voulez stocker sont confidentielles, utilisez un Secret plutôt qu'une ConfigMap. »
> — [kubernetes.io — ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

Voici un ConfigMap qui externalise l'URL de connexion à la base de données de `petclinic` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: petclinic-config
data:
  DATASOURCE_URL: "jdbc:h2:mem:petclinic"
  SPRING_PROFILES_ACTIVE: "h2"
```

### Rappel : qu'est-ce qu'une variable d'environnement ?

Avant d'aller plus loin, précisons ce concept — central dans tout ce qui suit. Une **variable
d'environnement** est une paire clé-valeur (ex. `SPRING_PROFILES_ACTIVE=h2`) transmise à un
programme **au moment où il démarre**, séparément de son code : une étiquette remise au programme
à son démarrage, pour reprendre l'analogie déjà vue en classe. Spring Boot (le framework de
`petclinic`) lit certaines de ces variables **automatiquement** pour se configurer, sans aucun
code supplémentaire — c'est exactement ce que fait `SPRING_PROFILES_ACTIVE`.

> 💡 **Le lien avec la semaine 7** : le Dockerfile de `petclinic` utilisait déjà `ENV
> SPRING_PROFILES_ACTIVE=production` ([Docker]({{% relref "/semaine-7/docker" %}}), section 3️⃣)
> pour définir cette même variable — mais **figée dans l'image**, identique pour tous les
> environnements. Ici, le Deployment déclare la variable **hors de l'image**, dans son YAML
> (`env` + `configMapKeyRef`, ci-dessous) : la même image `petclinic` peut ainsi recevoir une
> valeur différente selon l'environnement, sans jamais être reconstruite — et Kubernetes s'en
> charge pour **chaque** pod recréé, y compris ceux de l'auto-guérison (semaine 11), sans jamais
> retaper une commande.

### Monter un ConfigMap dans le Deployment de la semaine 11

Chaque valeur du ConfigMap est injectée **une à la fois**, comme variable d'environnement du
conteneur, avec `env` et `configMapKeyRef` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
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
          env:
            - name: DATASOURCE_URL
              valueFrom:
                configMapKeyRef:
                  name: petclinic-config
                  key: DATASOURCE_URL
            - name: SPRING_PROFILES_ACTIVE
              valueFrom:
                configMapKeyRef:
                  name: petclinic-config
                  key: SPRING_PROFILES_ACTIVE
```

> 💡 Rien ne change au niveau de l'image Docker elle-même : c'est strictement la même image
> `ghcr.io/<votre-compte>/petclinic:latest` qui tourne en développement et en production — seule
> la configuration **injectée au démarrage du conteneur** diffère, exactement le principe déjà vu
> à la semaine 7 avec le `CMD` remplaçable au `docker run`.

Appliquer d'abord le ConfigMap, puis le Deployment, et vérifier que la variable est bien arrivée
**dans** le conteneur :

```shell
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
kubectl exec -it <nom-du-pod> -- env | grep SPRING_PROFILES_ACTIVE
# SPRING_PROFILES_ACTIVE=h2
```

> ⚠️ **Piège fréquent** : modifier un ConfigMap déjà appliqué (`kubectl apply -f configmap.yaml`
> avec de nouvelles valeurs) ne redémarre **pas** automatiquement les pods qui l'utilisent — une
> variable d'environnement n'est lue qu'**au démarrage** du conteneur, jamais relue en cours de
> route. Pour forcer les pods existants à redémarrer et récupérer la nouvelle valeur :
> ```shell
> kubectl rollout restart deployment/petclinic
> ```

---

## 2️⃣ Données sensibles : Secret

### ConfigMap et Secret : même API, usage distinct

Un mot de passe de base de données ne doit jamais se retrouver dans un ConfigMap, visible en
clair par quiconque a accès à l'objet. Un `Secret` existe précisément pour ce cas :

> « Un Secret est un objet qui contient une petite quantité de données sensibles, comme un mot de
> passe, un jeton, ou une clé. [...] Les Secrets sont similaires aux ConfigMaps mais sont
> spécifiquement destinés à contenir des données confidentielles. »
> — [kubernetes.io — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

> ⚠️ **Un Secret n'est PAS chiffré par défaut.** La documentation officielle est explicite sur ce
> point :
> > « Les Secrets Kubernetes sont, par défaut, stockés non chiffrés dans le magasin de données
> > sous-jacent du serveur API (etcd). Quiconque a accès à l'API peut récupérer ou modifier un
> > Secret, tout comme quiconque a accès à etcd. »
> > — [kubernetes.io — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
>
> Les valeurs d'un Secret sont encodées en **base64** — un simple changement de représentation,
> pas un algorithme de chiffrement : n'importe qui peut décoder du base64 sans aucune clé
> (`echo "<valeur>" | base64 -d`). Un Secret Kubernetes, par défaut, ne garantit ni l'authenticité
> ni la confidentialité face à quelqu'un qui a déjà accès au cluster.

> 💡 **Alors, un Secret reste-t-il le bon choix pour un mot de passe ?** Oui — jamais un
> ConfigMap. La limite ci-dessus ne veut pas dire « n'utilisez pas Secret », mais « Secret seul ne
> suffit pas en production ». La documentation Kubernetes donne 4 étapes pour l'utiliser
> **en sécurité** :
>
> > « Pour utiliser les Secrets de façon sécuritaire, suivez au moins les étapes suivantes : (1)
> > Activez le chiffrement au repos (*Encryption at Rest*) pour les Secrets. (2) Activez ou
> > configurez des règles RBAC avec un accès à privilège minimal aux Secrets. (3) Restreignez
> > l'accès aux Secrets à des conteneurs spécifiques. (4) Envisagez d'utiliser des fournisseurs de
> > magasins de secrets externes. »
> > — [kubernetes.io — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
>
> Dans ce cours, l'étape (2) est déjà en place : chaque équipe travaille dans **son propre
> namespace** (semaine 11), ce qui limite déjà qui peut y lire un Secret. Les étapes (1) et (4)
> (chiffrement au repos, magasin de secrets externe) sont des renforcements supplémentaires, hors
> du programme de ce cours — mais bon à savoir si vous les rencontrez en entreprise.

Exemple de Secret pour le mot de passe de la base de données `petclinic` (la valeur est déjà
encodée en base64, pas chiffrée) :

> 💡 Le profil `h2` actuellement utilisé par `petclinic` ne requiert aucun mot de passe (base en
> mémoire, sans serveur séparé) — cet exemple illustre tout de même le mécanisme `Secret`, pour le
> jour où l'équipe passera à une base de données externe nécessitant une authentification.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: petclinic-db-secret
type: Opaque
data:
  DATASOURCE_PASSWORD: cGV0Y2xpbmlj  # base64 de "petclinic", à titre d'exemple
```

Monté dans le Deployment, toujours par variable d'environnement, mais avec `secretKeyRef` plutôt
que `configMapRef` :

```yaml
env:
  - name: DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: petclinic-db-secret
        key: DATASOURCE_PASSWORD
```

---

## 3️⃣ Logs : diagnostiquer un pod

Une fois l'application déployée, la première étape de tout débogage reste de lire ce qu'elle a
elle-même écrit sur sa sortie standard. La commande officielle pour cela est `kubectl logs` :

> « Affiche les logs d'un conteneur dans un pod ou une ressource spécifiée. Si le pod n'a qu'un
> seul conteneur, le nom du conteneur est optionnel. »
> — [kubernetes.io — kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)

```bash
# Logs récents d'un pod petclinic (nom obtenu via kubectl get pods)
kubectl logs petclinic-7d4f9c6b8d-x2k9p

# Suivre les logs en continu (-f, comme docker logs -f à la semaine 7)
kubectl logs -f petclinic-7d4f9c6b8d-x2k9p

# Logs du conteneur précédent, si le pod a redémarré après un crash
kubectl logs -p petclinic-7d4f9c6b8d-x2k9p

# Logs de TOUS les pods du Deployment en une seule commande
kubectl logs deployment/petclinic --all-pods=true
```

> 💡 **Le lien avec `docker logs`** (semaine 7) est direct : `kubectl logs` lit la même sortie
> standard/erreur du conteneur, simplement à travers l'API Kubernetes plutôt que par le démon
> Docker local. L'option `-p`/`--previous`, elle, répond à un problème propre à Kubernetes : un
> pod qui a redémarré (auto-guérison, semaine 11) remplace son conteneur — sans `-p`, vous liriez
> les logs du **nouveau** conteneur, pas de celui qui a réellement planté.

---

## 4️⃣ Exposer un Service à l'extérieur : Ingress

Un `Service` de type `ClusterIP` (semaine 11) reste invisible hors du cluster; un
`LoadBalancer` ouvre un port, mais sans routage par nom d'hôte ni par chemin d'URL. L'**Ingress**
résout précisément ce problème :

> « L'Ingress expose les routes HTTP et HTTPS de l'extérieur du cluster vers les services au sein
> du cluster. Le routage du trafic est contrôlé par des règles définies sur la ressource
> Ingress. »
> — [kubernetes.io — Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

> « Un Ingress peut être configuré pour donner aux Services des URL accessibles de l'extérieur,
> répartir la charge du trafic, terminer le SSL/TLS, et offrir un hébergement virtuel basé sur le
> nom. »
> — [kubernetes.io — Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

> ⚠️ Un Ingress ne fait **rien** à lui seul : il a besoin d'un **contrôleur Ingress** installé
> sur le cluster pour être réellement appliqué.
> > « Vous devez avoir un contrôleur Ingress pour satisfaire un Ingress. Créer seulement une
> > ressource Ingress n'a aucun effet. »
> > — [kubernetes.io — Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
>
> La documentation Kubernetes note aussi que le projet recommande désormais l'API **Gateway**
> plutôt qu'Ingress pour les nouveaux projets, l'API Ingress étant « gelée » (plus aucune
> évolution, mais toujours officiellement supportée) — ce cours couvre Ingress parce qu'il reste
> la ressource la plus répandue en pratique, mais gardez ce détail en tête si vous explorez plus
> loin que ce qui est couvert ici.

> 💡 **Pour aller plus loin (hors programme) : à quoi ressemble l'API Gateway ?** Même objectif
> qu'Ingress (router le trafic HTTP externe vers des Services internes), mais découpé en **3**
> ressources distinctes plutôt qu'une seule `Ingress` : `GatewayClass` (quel contrôleur gère le
> trafic — équivalent du contrôleur Ingress installé ci-dessous), `Gateway` (l'infrastructure
> réseau concrète — équivalent du `LoadBalancer` et de son IP publique), et `HTTPRoute` (les
> règles de routage elles-mêmes — équivalent des `rules`/`paths` d'un Ingress) :
>
> ```yaml
> apiVersion: gateway.networking.k8s.io/v1
> kind: HTTPRoute
> metadata:
>   name: petclinic-route
> spec:
>   parentRefs:
>     - name: petclinic-gateway   # référence un objet Gateway existant
>   rules:
>     - backendRefs:
>         - name: petclinic-service
>           port: 9966
> ```
>
> — [kubernetes.io — Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)

### Installer le contrôleur Ingress et obtenir un nom de domaine, avec Helm

```shell
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=petclinic-etu1234
```

- `helm upgrade --install` : installe si absent, met à jour sinon — une seule commande réutilisable.
- `ingress-nginx ingress-nginx` : nom de la release, puis nom du chart.
- `--repo ...` : dépôt officiel du chart (pas besoin de le cloner).
- `--namespace ingress-nginx --create-namespace` : installe dans son propre namespace, créé s'il n'existe pas.
- `--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=petclinic-etu1234` :
  ajoute l'annotation Azure qui publie un nom de domaine gratuit. Les `\.` empêchent Helm
  d'interpréter les points de cette clé comme des séparateurs de structure (il la traite comme un
  seul nom plutôt que de la découper).

Cette commande crée un `Service ingress-nginx-controller` (nom de la release + `-controller`) de
type `LoadBalancer`. Vérifier qu'une IP publique a été attribuée :

```shell
kubectl get service ingress-nginx-controller -n ingress-nginx
```

→ affiche l'`EXTERNAL-IP`, mais **pas** le nom de domaine (FQDN) — c'est une ressource Azure,
pas Kubernetes. Récupérer le FQDN généré :

```shell
az network public-ip list \
  --query "[?dnsSettings.domainNameLabel=='petclinic-etu1234'].dnsSettings.fqdn" -o tsv
```

→ ex. `petclinic-etu1234.canadacentral.cloudapp.azure.com` — à mettre comme `host:` dans
l'Ingress ci-dessous.

Exemple minimal pour exposer le Service `petclinic-service` (semaine 11) via ce nom d'hôte :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: petclinic-ingress
spec:
  ingressClassName: nginx   # précise quel contrôleur traite cet Ingress
  rules:
    - host: petclinic-etu1234.canadacentral.cloudapp.azure.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: petclinic-service
                port:
                  number: 9966
```

> 💡 `ingressClassName: nginx` indique explicitement quel contrôleur Ingress doit traiter cette
> ressource (ici, celui installé plus haut avec Helm, dont la classe par défaut est `nginx`) —
> utile dès qu'il y a plus d'un contrôleur Ingress dans le cluster, ou pour ne pas dépendre d'une
> classe « par défaut » implicite.

> 💡 Pour un vrai nom de domaine personnalisé (ex. `petclinic.monentreprise.com` plutôt que le
> suffixe `cloudapp.azure.com`), il faudrait un domaine enregistré séparément (chez un registraire)
> et une zone **Azure DNS** pour le pointer vers cette même IP publique — hors du cadre de ce
> cours, mais bon à savoir si vous déployez un jour un vrai site en production.

> ⚠️ **Dépannage : erreur 504 à travers l'Ingress**. Le *load balancer* Azure sonde
> régulièrement le `Service ingress-nginx-controller` pour savoir s'il peut lui envoyer du
> trafic ; si cette sonde (*health probe*) échoue, Azure considère le backend en panne et coupe
> le trafic, ce qui produit un **504** côté navigateur même si l'application fonctionne. Le
> correctif est l'annotation `service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path`,
> qui indique le chemin à sonder :
>
> ```shell
> kubectl annotate service ingress-nginx-controller -n ingress-nginx \
>   service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path="/healthz" --overwrite
> ```
>
> - `kubectl annotate service ... -n ingress-nginx` : modifie une annotation sur le Service déjà
>   créé par Helm (pas besoin de réinstaller).
> - `azure-load-balancer-health-probe-request-path="/healthz"` : le chemin que le load balancer
>   Azure doit appeler pour vérifier la santé du backend — `/healthz` est l'endpoint exposé par
>   le contrôleur ingress-nginx à cette fin.
> - `--overwrite` : requis par `kubectl annotate` dès que l'annotation existe déjà (même mise à
>   la même valeur), sinon la commande refuse de l'écraser.
>
> — [cloud-provider-azure — LoadBalancer annotations](https://cloud-provider-azure.sigs.k8s.io/topics/loadbalancer/)

---

## 📊 ConfigMap vs Secret : quand utiliser lequel

| Donnée | ConfigMap | Secret |
|---|---|---|
| URL de la base de données | ✅ | — |
| Nom de profil Spring (`h2`, `mysql`, `postgres`) | ✅ | — |
| Mot de passe de la base de données | ❌ | ✅ |
| Jeton d'API externe | ❌ | ✅ |
| Donnée confidentielle, mais chiffrement requis en production | ❌ (ne chiffre rien) | ⚠️ seulement combiné à *Encryption at Rest* + RBAC |

---

<details>
<summary>🤔 Testez-vous</summary>

Un coéquipier affirme : « On a mis le mot de passe de la base de données dans un Secret
Kubernetes au lieu d'un ConfigMap, donc il est maintenant chiffré et personne ne peut le lire sans
autorisation. » Qu'est-ce qui est vrai dans cette affirmation, et qu'est-ce qui ne l'est pas ?

**Réponse** : c'est vrai qu'un Secret est le bon choix (plutôt qu'un ConfigMap) pour une donnée
confidentielle, et que l'API Kubernetes traite les Secrets différemment (ex. ils ne sont pas
affichés par défaut par certaines commandes). Mais c'est faux de dire qu'il est « chiffré » : la
documentation officielle précise que les Secrets sont stockés **non chiffrés** dans etcd par
défaut, et que leurs valeurs ne sont qu'**encodées en base64** — réversible par quiconque sans
aucune clé. Une protection réelle exige d'activer explicitement le chiffrement au repos et des
règles RBAC restreintes, en plus d'utiliser un Secret plutôt qu'un ConfigMap.
</details>

---

## 🔎 Pour aller plus loin (hors programme) : Azure Key Vault

Le point faible identifié plus haut — un Secret non chiffré, protégé seulement par RBAC — a une
réponse spécifique à Azure. Le fournisseur **Azure Key Vault pour Secrets Store CSI Driver**
permet à un pod AKS de lire une valeur directement depuis un **Key Vault** Azure (chiffré,
journalisé, avec ses propres permissions), plutôt que depuis un `Secret` Kubernetes stocké dans
etcd :

> « Le fournisseur Azure Key Vault pour Secrets Store Container Storage Interface (CSI) Driver
> permet l'intégration d'un Azure Key Vault comme magasin de secrets avec un cluster Azure
> Kubernetes Service (AKS), via un volume CSI. »
> — [Microsoft Learn — Azure Key Vault provider for Secrets Store CSI Driver](https://learn.microsoft.com/azure/aks/csi-secrets-store-driver)

### Activer le fournisseur sur un cluster AKS existant

```bash
az aks enable-addons \
  --addons azure-keyvault-secrets-provider \
  --name <nom-cluster> \
  --resource-group <nom-groupe-ressources>
```

> 💡 Cette commande crée automatiquement une **identité managée** (ex.
> `azurekeyvaultsecretsprovider-<nom-cluster>`) que le cluster utilise pour s'authentifier auprès
> du Key Vault — sans mot de passe stocké nulle part.

### Créer le Key Vault et y déposer un secret

```bash
az keyvault create --name <nom-coffre> --resource-group <nom-groupe-ressources> --enable-rbac-authorization
az keyvault secret set --vault-name <nom-coffre> --name DbPassword --value "<mot-de-passe-reel>"
```

Il faut ensuite donner à l'identité managée créée plus haut la permission de **lire** ce secret
(rôle Azure RBAC `Key Vault Secrets User`, limité au seul Key Vault concerné), avant que le pod
puisse y accéder.

### Monter le secret dans un pod, et le synchroniser comme un vrai `Secret` Kubernetes

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: petclinic-keyvault
spec:
  provider: azure
  secretObjects:                      # optionnel : synchronise la valeur lue dans un vrai Secret Kubernetes
    - secretName: petclinic-db-secret
      type: Opaque
      data:
        - objectName: DbPassword
          key: DATASOURCE_PASSWORD
  parameters:
    keyvaultName: "<nom-coffre>"
    tenantId: "<tenant-id-azure>"
    objects: |
      array:
        - |
          objectName: DbPassword
          objectType: secret
```

```yaml
# Dans le Deployment : le volume CSI déclenche la lecture ET la synchronisation ci-dessus
volumes:
  - name: petclinic-kv
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "petclinic-keyvault"
```

> ⚠️ D'après la documentation officielle : « Les secrets ne se synchronisent qu'au moment où un
> pod montant ces secrets démarre. » — le volume CSI (monté dans le pod) est donc **obligatoire**
> pour que la synchronisation avec un `Secret` Kubernetes se produise; on ne peut pas s'appuyer
> uniquement sur `secretObjects` sans jamais monter le volume.
> — [secrets-store-csi-driver.sigs.k8s.io — Sync as Kubernetes Secret](https://secrets-store-csi-driver.sigs.k8s.io/topics/sync-as-kubernetes-secret.html)

Une fois synchronisé, le `Secret` `petclinic-db-secret` ci-dessus s'utilise **exactement** comme
celui de la section 2️⃣ (même `secretKeyRef` dans le Deployment) — Key Vault ne change que
**l'origine** de la valeur, jamais la façon de la consommer dans l'application.

---

- [kubernetes.io — ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [kubernetes.io — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [kubernetes.io — kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
- [kubernetes.io — Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Microsoft Learn — Azure Key Vault provider for Secrets Store CSI Driver](https://learn.microsoft.com/azure/aks/csi-secrets-store-driver)
- [secrets-store-csi-driver.sigs.k8s.io — Sync as Kubernetes Secret](https://secrets-store-csi-driver.sigs.k8s.io/topics/sync-as-kubernetes-secret.html)
- [Microsoft Learn — Use a static IP with a load balancer in AKS](https://learn.microsoft.com/azure/aks/static-ip)

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Helm et déploiement continu]({{% relref "/semaine-12/helm-deploiement-continu" %}})
pour voir comment empaqueter ce Deployment, ce Service, ce ConfigMap et ce Secret dans un seul
chart réutilisable, et les déployer sans interruption de service.
{{% /notice %}}
