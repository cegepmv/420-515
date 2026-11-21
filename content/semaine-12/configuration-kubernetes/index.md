+++
title = "Configuration, secrets et exposition : ConfigMap, Secret, logs, Ingress"
weight = 1
draft = false
+++

Tout ce qui suit est fondé sur la documentation officielle de Kubernetes
(`kubernetes.io`). Les exemples YAML réutilisent le Deployment/Service `petclinic` introduit à la
semaine 11 ([Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}})).

> 💡 **Rappel (semaine 11)** : aucun des manifestes ci-dessous n'indique de champ
> `metadata.namespace` explicite — exactement comme le Deployment et le Service de la semaine 11.
> Ils s'appliquent donc tous dans le namespace **courant de votre contexte**
> (`etu1234` ou `equipe-NN`, selon la façon dont vous l'avez fixé avec
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
  DATASOURCE_URL: "jdbc:mysql://petclinic-db:3306/petclinic"
  SPRING_PROFILES_ACTIVE: "mysql"
```

### Rappel : qu'est-ce qu'une variable d'environnement ?

Avant d'aller plus loin, précisons ce concept — central dans tout ce qui suit. Une **variable
d'environnement** est une paire clé-valeur (ex. `SPRING_PROFILES_ACTIVE=mysql`) transmise à un
programme **au moment où il démarre**, séparément de son code : une étiquette remise au programme
à son démarrage, pour reprendre l'analogie déjà vue en classe. Spring Boot (le framework de
`petclinic`) lit certaines de ces variables **automatiquement** pour se configurer, sans aucun
code supplémentaire — c'est exactement ce que fait `SPRING_PROFILES_ACTIVE`.

> 💡 **Le lien avec la semaine 7** : `docker run -e SPRING_PROFILES_ACTIVE=mysql petclinic`
> faisait déjà exactement ça — injecter une variable d'environnement dans le conteneur au
> démarrage, sans modifier l'image. Ici, c'est le Deployment qui déclare, une fois pour toutes
> dans son YAML, quelles variables injecter (`env` + `configMapKeyRef`, ci-dessous) — et
> Kubernetes s'en charge pour **chaque** pod recréé, y compris ceux de l'auto-guérison
> (semaine 11), sans jamais retaper la commande.

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
# SPRING_PROFILES_ACTIVE=mysql
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
> (`echo "<valeur>" | base64 -d`). C'est exactement la distinction déjà posée à la semaine 10
> entre **signer** et **chiffrer** un fichier : signer un paquet Debian garantit son authenticité,
> pas sa confidentialité ([Distribution de mises à jour et sécurité]({{% relref "/semaine-10/distribution-maj-securite" %}}))
> — un Secret Kubernetes, par défaut, ne garantit ni l'une ni l'autre face à quelqu'un qui a déjà
> accès au cluster. La documentation recommande d'ailleurs explicitement d'activer un chiffrement
> au repos (*Encryption at Rest*) et des règles RBAC restrictives pour un usage réellement sûr en
> production.

Exemple de Secret pour le mot de passe de la base de données `petclinic` (la valeur est déjà
encodée en base64, pas chiffrée) :

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

Exemple minimal pour exposer le Service `petclinic` (semaine 11) via un nom d'hôte :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: petclinic-ingress
spec:
  rules:
    - host: petclinic.votre-domaine.exemple
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: petclinic
                port:
                  number: 9966
```

---

## 📊 ConfigMap vs Secret : quand utiliser lequel

| Donnée | ConfigMap | Secret |
|---|---|---|
| URL de la base de données | ✅ | — |
| Nom de profil Spring (`mysql`, `postgres`) | ✅ | — |
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

## 📚 Références

- [kubernetes.io — ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [kubernetes.io — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [kubernetes.io — kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
- [kubernetes.io — Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Helm et déploiement continu]({{% relref "/semaine-12/helm-deploiement-continu" %}})
pour voir comment empaqueter ce Deployment, ce Service, ce ConfigMap et ce Secret dans un seul
chart réutilisable, et les déployer sans interruption de service.
{{% /notice %}}
