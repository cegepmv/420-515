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
  namespace: equipe-07
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
          image: ghcr.io/equipe-07/petclinic:v1.4.0
          ports:
            - containerPort: 9966
---
apiVersion: v1
kind: Service
metadata:
  name: petclinic-service
  namespace: equipe-07
spec:
  type: ClusterIP
  selector:
    app: petclinic
  ports:
    - protocol: TCP
      port: 9966
      targetPort: 9966
```

1. À quoi sert le champ `namespace: equipe-07` sur ces deux manifestes ? Dans ce cours, chaque
   équipe a son propre cluster AKS — ce champ est-il donc inutile ici ? Expliquez dans quelle
   situation (pas nécessairement celle de ce cours) ce champ deviendrait indispensable.
2. Un des trois pods créés par ce Deployment plante. Combien de temps, selon vous, le Service
   reste-t-il fonctionnel, et pourquoi n'a-t-il besoin d'aucune reconfiguration manuelle ?
3. Ce Service est de type `ClusterIP`. Un coéquipier essaie d'y accéder depuis son navigateur, à la
   maison, et ça ne fonctionne pas. Expliquez pourquoi, et quel changement de **type** de Service
   serait nécessaire pour que ça fonctionne (en supposant que le cluster utilisé le permette).
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
2. Une équipe doit reprendre son travail le lendemain matin, après avoir arrêté son cluster la
   veille. Avec quelle commande redémarre-t-elle le cluster, et combien de temps doit-elle prévoir
   avant de pouvoir relancer immédiatement un arrêt si nécessaire ?
3. Une équipe a déjà dépensé 95 $ de son crédit Azure (sur 100 $) à mi-session, en laissant son
   cluster actif en continu depuis le début. Quelle pratique, décrite dans les notes, aurait permis
   d'éviter cette situation ?
4. Un étudiant exécute `az aks stop`, puis tente immédiatement `az aks start` quelques secondes
   plus tard. Que risque-t-il, selon la documentation officielle ?

---

<details>
<summary>❓ Corrigé — Partie 1</summary>

1. Le `namespace` cloisonne les ressources à l'intérieur d'un cluster : les noms de ressources
   (`petclinic`, `petclinic-service`) ne sont uniques **qu'à l'intérieur** d'un namespace, pas à
   l'échelle du cluster entier. Dans ce cours, puisque chaque équipe possède déjà son propre
   cluster, ce champ n'évite aucun conflit réel — personne d'autre ne travaille sur votre cluster.
   Il deviendrait indispensable dès que **plusieurs équipes ou projets** partageraient le **même**
   cluster (la situation normale en entreprise) : sans cette isolation, deux projets qui déploient
   chacun un Deployment `petclinic` dans `default` entreraient directement en conflit.
2. Le Service reste fonctionnel en tout temps, sans aucune reconfiguration : le Deployment détecte
   immédiatement que l'état réel (2 pods vivants) ne correspond plus à l'état désiré
   (`replicas: 3`) et en recrée un nouveau (auto-guérison). Le `selector` du Service cible
   l'étiquette `app: petclinic`, pas des pods précis par IP ou par nom — dès que le pod de
   remplacement porte la même étiquette, il est automatiquement inclus dans la répartition de
   charge.
3. `ClusterIP` n'expose le Service qu'**à l'intérieur** du cluster — aucune IP externe n'est
   attribuée. Pour qu'un navigateur, depuis l'extérieur du cluster (la maison du coéquipier),
   puisse s'y connecter, il faudrait un type `LoadBalancer` (ou un Ingress, vu à la semaine 12),
   qui expose le Service via une adresse externe/un load balancer.
4. Le Service ne trouverait plus **aucun** pod à cibler : son `selector` (`app: petclinic`) ne
   correspondrait plus aux nouvelles étiquettes des pods créés par le Deployment modifié. Le
   Service existerait toujours, mais router aucune requête vers aucun pod (liste de endpoints
   vide) — un bris de service silencieux, sans erreur explicite à l'application du YAML.
</details>

<details>
<summary>❓ Corrigé — Partie 2</summary>

1. Il devrait arrêter son cluster AKS avec `az aks stop --name <nom-cluster> --resource-group
   <nom-groupe>`, pour éviter que la facturation à l'heure ne continue de s'appliquer pendant la
   nuit, même si personne ne travaille activement dessus.
2. `az aks start --name <nom-cluster> --resource-group <nom-groupe>`. La documentation officielle
   recommande d'attendre 15 à 30 minutes après un arrêt avant de redémarrer le cluster, le temps
   que l'opération d'arrêt se termine complètement.
3. Arrêter systématiquement le cluster (`az aks stop`) à la fin de **chaque** session de travail,
   plutôt que de le laisser tourner en continu — un cluster actif est facturé à l'heure, que
   quelqu'un l'utilise ou non.
4. Il risque de perturber l'arrêt en cours : la documentation officielle précise qu'il faut
   attendre 15 à 30 minutes après un `az aks stop` avant de redémarrer, le temps que l'arrêt se
   termine complètement.
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
