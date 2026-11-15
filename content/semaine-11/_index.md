+++
title = "Semaine 11 — Kubernetes 1/2 : namespace, Deployment, Service"
type = "chapter"
weight = 12
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Expliquer pourquoi Docker seul (semaine 7) ne résout pas le problème de **faire tourner** une
  application en production : redémarrage automatique, répartition de charge entre plusieurs
  copies, mise à l'échelle.
- Déployer un conteneur sur **Azure Container Instances (ACI)** et constater par vous-même ses
  limites (adresse qui change au redémarrage, aucune copie automatique) avant de voir comment
  Kubernetes les résout.
- Décrire, dans leurs grandes lignes, les rôles du **control plane** et des **nœuds** dans un
  cluster Kubernetes.
- Expliquer ce qu'est un **namespace** et pourquoi ce cloisonnement est utile dans un cluster
  Kubernetes, même si votre équipe travaille seule sur le sien pendant ce cours.
- Lire et écrire un manifeste **Deployment** simple (réplicas, image, port), et expliquer ce que
  signifie « auto-guérison » (*self-healing*) dans ce contexte.
- Lire et écrire un manifeste **Service** simple, et expliquer la différence entre `ClusterIP` et
  `LoadBalancer`, ainsi que le principe de répartition de charge entre les pods d'un Deployment.
- Expliquer pourquoi le cours utilise un cluster managé **Azure Kubernetes Service (AKS)**, créer
  et nommer correctement le sien à partir de son crédit étudiant Azure, et savoir
  l'arrêter/redémarrer pour ne pas épuiser ce crédit avant la fin de la session.

---

## 🔍 Accroche

> « Votre application tourne dans un conteneur Docker (semaine 7). Très bien. Mais qui la relance
> à 3h du matin quand le conteneur plante? Qui s'assure que deux ou trois copies tournent en même
> temps pour absorber la charge, et que le trafic est réparti entre elles sans que personne n'ait
> à y penser manuellement? Un `docker run` ne répond à aucune de ces questions : une fois le
> conteneur arrêté, il reste arrêté, jusqu'à ce qu'un humain relance la commande. C'est exactement
> le problème que **Kubernetes** a été conçu pour résoudre — et c'est aussi, cette semaine, le
> prétexte pour découvrir votre terrain de jeu pour le reste de la session : un cluster
> **Azure Kubernetes Service**, financé par votre crédit étudiant, qu'il faudra apprendre à gérer
> intelligemment pour ne pas l'épuiser avant la fin du cours. »
