+++
title = "Exercice : Kubernetes 1/2"
weight = 4
draft = false
+++

## 🎯 Objectif

Appliquer les concepts vus dans
[Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}}) et
[Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}}) à des
situations concrètes.

---

## 🧩 Partie 1 — Lire un Deployment et un Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
  namespace: etu1234
spec:
  replicas: 3
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
          image: ghcr.io/etu1234/petclinic:v1.4.0
          ports:
            - containerPort: 9966
---
apiVersion: v1
kind: Service
metadata:
  name: petclinic-service
  namespace: etu1234
spec:
  type: ClusterIP
  selector:
    app: petclinic
  ports:
    - protocol: TCP
      port: 9966
      targetPort: 9966
```

1. À quoi sert le champ `namespace: etu1234` sur ces deux manifestes ? Dans ce cours, chaque
   étudiant a son propre cluster AKS — ce champ est-il donc inutile ici ? Expliquez dans quelle
   situation (pas nécessairement celle de ce cours) ce champ deviendrait indispensable.
2. Un des trois pods créés par ce Deployment plante. Combien de temps, selon vous, le Service
   reste-t-il fonctionnel, et pourquoi n'a-t-il besoin d'aucune reconfiguration manuelle ?
3. Ce Service est de type `ClusterIP`. Un camarade de classe essaie d'y accéder depuis son
   navigateur, à la maison, et ça ne fonctionne pas. Expliquez pourquoi, et quel changement de
   **type** de Service serait nécessaire pour que ça fonctionne (en supposant que le cluster
   utilisé le permette).
4. Le `selector` du Service et les `matchLabels` du Deployment utilisent la même valeur
   (`app: petclinic`). Que se passerait-il si on changeait l'étiquette dans `template.metadata.labels`
   du Deployment sans changer le `selector` du Service ?

---

## 🧩 Partie 2 — Gérer son crédit Azure

Pour chacune des situations suivantes, indiquez la bonne action à prendre (et la commande, le cas
échéant), en justifiant à l'aide de
[Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}}) :

1. Un étudiant termine sa session de travail sur AKS pour la soirée. Que devrait-il faire avant de
   fermer son ordinateur, et avec quelle commande ?
2. Un étudiant doit reprendre son travail le lendemain matin, après avoir arrêté son cluster la
   veille. Avec quelle commande redémarre-t-il le cluster, et combien de temps doit-il prévoir
   avant de pouvoir relancer immédiatement un arrêt si nécessaire ?
3. Un étudiant a déjà dépensé 95 $ de son crédit Azure (sur 100 $) à mi-session, en laissant son
   cluster actif en continu depuis le début. Quelle pratique, décrite dans les notes, aurait permis
   d'éviter cette situation ?
4. Un étudiant exécute `az aks stop`, puis tente immédiatement `az aks start` quelques secondes
   plus tard. Que risque-t-il, selon la documentation officielle ?

---

<details>
<summary>❓ Corrigé — Partie 1</summary>

1. Le `namespace` cloisonne les ressources à l'intérieur d'un cluster : la documentation
   officielle est précise sur ce point — « les noms des ressources doivent être uniques dans un
   namespace, mais pas à travers les namespaces ». Concrètement, cela veut dire que
   `kubectl get deployment petclinic` ne renvoie **qu'un seul** résultat possible à l'intérieur
   d'un namespace donné, mais que rien n'empêche l'existence d'un autre Deployment nommé
   `petclinic` dans un namespace voisin, sur le **même** cluster, sans le moindre conflit — un peu
   comme deux immeubles différents peuvent chacun avoir un appartement « 301 » sans confusion,
   parce que l'adresse complète inclut le nom de l'immeuble.

   Dans ce cours, ce champ n'évite **aucun conflit réel**, puisque chaque étudiant possède son
   propre cluster, à lui seul : il n'y a personne d'autre avec qui partager `etu1234`, donc même en
   supprimant complètement le champ `namespace` (et en travaillant dans `default`), rien de
   fâcheux ne se produirait techniquement.

   Ce champ deviendrait **indispensable** dès que plusieurs équipes ou projets partageraient le
   **même** cluster — la situation normale en entreprise. Exemple concret : imaginez un cluster
   d'entreprise partagé par l'équipe « Facturation » et l'équipe « Inventaire ». Si les deux
   équipes déploient chacune, indépendamment, un Deployment nommé `api` et un Service nommé
   `api-service` dans le namespace `default`, la seconde équipe à exécuter `kubectl apply`
   **écrasera silencieusement** les ressources de la première (même nom = même ressource aux yeux
   de l'API Kubernetes) : les pods de la Facturation disparaissent, remplacés par ceux de
   l'Inventaire, sans le moindre message d'erreur. Avec des namespaces distincts
   (`facturation` et `inventaire`), les deux équipes peuvent nommer leurs ressources `api` et
   `api-service` sans jamais se marcher sur les pieds — exactement le scénario décrit par la
   documentation officielle : « les namespaces sont un moyen de diviser les ressources du cluster
   entre plusieurs utilisateurs ».

2. Le Service reste fonctionnel **en tout temps**, sans aucune reconfiguration manuelle — c'est
   précisément le mécanisme de réconciliation vu en section 1️⃣ de
   [Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}}). Déroulons la
   séquence concrète : (1) un pod `petclinic-7d4f9c6b8d-x2k9p` plante (ex. erreur mémoire); (2) le
   `kube-controller-manager`, qui compare en continu l'état réel à l'état désiré, détecte en
   quelques secondes que seulement 2 pods tournent au lieu des 3 demandés (`replicas: 3`); (3) il
   demande au `kube-scheduler` de placer un nouveau pod sur un nœud disponible; (4) ce nouveau pod,
   par exemple nommé `petclinic-7d4f9c6b8d-m8w2q`, reçoit **automatiquement** l'étiquette
   `app: petclinic` héritée du `template.metadata.labels` du Deployment — jamais écrite à la main.

   Pendant toute cette séquence (quelques secondes à quelques dizaines de secondes), le Service
   `petclinic-service` n'a **rien à faire** : son `selector` (`app: petclinic`) ne cible pas des
   pods précis par IP ou par nom — qui changeraient de toute façon à chaque remplacement — mais une
   **étiquette**. Kubernetes maintient en coulisses une liste d'« Endpoints » (les IP des pods
   portant l'étiquette ciblée) mise à jour automatiquement : dès que le nouveau pod devient
   `Ready`, son IP est ajoutée à cette liste, et le Service recommence à lui envoyer une partie du
   trafic, sans redémarrage ni modification de son propre manifeste YAML.

3. `ClusterIP` — la valeur par défaut d'un Service — n'expose le Service qu'**à l'intérieur** du
   cluster : aucune adresse IP publique, aucun port ouvert vers Internet. Concrètement, un
   `ClusterIP` ressemble à `10.0.123.45` — une adresse qui n'existe que dans le réseau privé du
   cluster Azure, totalement injoignable depuis la maison du camarade de classe, même en
   connaissant cette adresse par cœur : son fournisseur Internet ne sait tout simplement pas
   comment router un paquet vers `10.0.123.45`, qui n'a de sens que *dans* le réseau interne
   d'Azure.

   Pour que l'accès fonctionne depuis l'extérieur, il faudrait changer `spec.type` à
   `LoadBalancer` : Azure (le fournisseur de cloud sous-jacent, via le
   `cloud-controller-manager` — voir [Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}}))
   provisionnerait alors un **équilibreur de charge réel**, avec une adresse IP **publique** (ex.
   `20.123.45.67`), cette fois joignable depuis n'importe quel navigateur sur Internet — exactement
   la configuration utilisée pour `petclinic` dans
   [Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}}).

4. Le Service **perdrait tous ses pods cibles**, d'un coup. Exemple concret : si
   `template.metadata.labels` passe de `app: petclinic` à `app: petclinic-v2` (par exemple lors
   d'un renommage maladroit), les trois nouveaux pods recréés par le Deployment porteront
   l'étiquette `app: petclinic-v2` — mais le `selector` du Service, lui, continue de chercher
   `app: petclinic`. Résultat : la liste d'Endpoints du Service devient **vide** — `kubectl get
   endpoints petclinic-service` afficherait `<none>` — alors que `kubectl get pods` montrerait
   bel et bien 3 pods `Running`. Le Service existe toujours, répond toujours sur son port, mais ne
   route plus **aucune** requête vers aucun pod : un navigateur qui tente d'y accéder resterait en
   attente indéfiniment, sans message d'erreur explicite du côté de Kubernetes — un bris de
   service silencieux, souvent difficile à diagnostiquer pour qui ne pense pas immédiatement à
   comparer `selector` et `labels`.
</details>

<details>
<summary>❓ Corrigé — Partie 2</summary>

1. Il devrait arrêter son cluster AKS avec `az aks stop --name <nom-cluster> --resource-group
   <nom-groupe>`. Concrètement : un cluster AKS actif facture en continu les machines virtuelles
   (les **nœuds**) qui le composent, à l'heure, **même si zéro pod n'y tourne activement** — un
   cluster oublié allumé pendant un week-end complet (environ 60 heures) peut facilement
   représenter plusieurs dollars de crédit étudiant, dépensés pour rien. `az aks stop` désalloue
   complètement ces machines virtuelles : la facturation de calcul s'arrête, et seul le stockage
   résiduel (minime) continue d'être facturé.
2. `az aks start --name <nom-cluster> --resource-group <nom-groupe>`. La documentation officielle
   recommande d'attendre **15 à 30 minutes** après un arrêt avant de redémarrer le cluster — le
   temps que l'opération d'arrêt (désallocation des machines virtuelles) se termine complètement
   côté Azure. Démarrer immédiatement après un arrêt (sans attendre) est précisément le scénario
   couvert à la question 4.
3. Arrêter systématiquement le cluster (`az aks stop`) à la fin de **chaque** session de travail,
   plutôt que de le laisser tourner en continu entre deux séances. Exemple concret : avec 100 $ de
   crédit pour toute la session, et en supposant un coût approximatif de l'ordre de 0,10 $–0,15 $
   l'heure pour un petit cluster à un seul nœud, laisser tourner le cluster 24 h/24 pendant
   7 semaines (environ 1 175 heures) épuiserait le crédit bien avant la fin de la session — alors
   que l'arrêter chaque soir et chaque fin de semaine (en ne le laissant actif que pendant les
   quelques heures réellement travaillées par semaine) réduit cette facture d'un facteur de 5 à 10.
4. Il risque que la commande `az aks start` échoue ou se comporte de façon imprévisible, parce que
   l'opération d'arrêt précédente (`az aks stop`) n'est pas encore terminée côté Azure : la
   documentation officielle précise qu'il faut attendre 15 à 30 minutes entre les deux commandes.
   Dans les faits, Azure retourne généralement une erreur explicite indiquant qu'une opération est
   déjà en cours sur la ressource — la bonne pratique reste donc d'attendre plutôt que de réessayer
   en boucle immédiatement.
</details>

---

## ✅ Auto-vérification

- [ ] Je peux expliquer à quoi sert un namespace, et dans quelle situation il devient nécessaire.
- [ ] Je peux expliquer ce que signifie « auto-guérison » pour un Deployment, concrètement.
- [ ] Je connais la différence entre `ClusterIP` et `LoadBalancer`, et je sais laquelle choisir
      selon qui doit pouvoir accéder au Service.
- [ ] Je connais les commandes pour arrêter et redémarrer un cluster AKS, et je sais pourquoi
      c'est important pour préserver mon crédit étudiant.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[activité pratique]({{% relref "/semaine-11/activite-pratique" %}}) pour déployer
réellement `petclinic` sur votre cluster AKS.
{{% /notice %}}
