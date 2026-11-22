+++
title = "Semaine 12 — Kubernetes 2/2 : ConfigMap, Secrets, Ingress et Helm"
type = "chapter"
weight = 13
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Externaliser un paramètre de configuration de l'application legacy hors de l'image Docker, à
  l'aide d'un **ConfigMap** Kubernetes.
- Gérer une donnée sensible (mot de passe, jeton) avec un **Secret** Kubernetes, et expliquer
  précisément ce que l'encodage utilisé protège — et ce qu'il ne protège **pas**.
- Consulter les **logs** d'un pod en cours d'exécution ou déjà terminé, pour diagnostiquer un
  problème.
- Exposer un Service existant (semaine 11) à l'extérieur du cluster via un nom d'hôte, à l'aide
  d'un **Ingress**.
- Empaqueter un déploiement complet (Deployment, Service, ConfigMap, Secret) dans un **chart
  Helm**, paramétré par un fichier `values.yaml`.
- Expliquer le mécanisme du **rolling update**, la stratégie par défaut d'un Deployment
  Kubernetes, et la situer par rapport aux alternatives (*blue-green*, *canary*).
- **Déployer réellement** une mise à jour de l'application avec `helm upgrade`, sans interruption
  de service.

---

## 🔍 Accroche

« La semaine 11 vous a donné de quoi faire tourner l'application legacy sur un cluster
Kubernetes : un namespace pour l'isoler, un Deployment pour la garder en vie, un Service pour y
accéder. Mais un `Deployment` tel quel a trois angles morts très concrets. D'abord, l'URL de votre
base de données et le mot de passe qui va avec sont, pour l'instant, soit codés en dur dans
l'image Docker (ce que la semaine 7 déconseillait déjà), soit absents du YAML que vous avez écrit
à la main. Ensuite, si un pod plante, vous n'avez, pour l'instant, aucun moyen de savoir
*pourquoi* sans redémarrer l'investigation à zéro. Enfin, un `Service` de type `ClusterIP` reste
invisible depuis l'extérieur du cluster — et un `LoadBalancer` ouvre un port, mais sans nom
d'hôte ni routage par chemin. Cette semaine ferme ces trois angles morts, puis règle un quatrième
problème, plus structurel : copier-coller le même YAML pour chaque équipe ou chaque mise à jour
ne passe pas à l'échelle. **Helm** répond à ce dernier problème — et sert de point d'appui pour la
remise la plus importante de la session : déployer une mise à jour réelle de
l'application, via Helm, **sans interruption de service**. »
