+++
title = "Activité pratique"
weight = 5
draft = false
+++

## 🎯 Activité pratique

En équipe, en réinvestissant directement
[Kubernetes fondamentaux]({{% relref "/semaine-11/kubernetes-fondamentaux" %}}) et
[Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}}) :

> 💡 Cette activité n'est **pas notée**. Elle sert à préparer concrètement le terrain pour la
> semaine 12, où vous ajouterez configuration, Ingress et déploiement continu via Helm à partir de
> ce que vous aurez construit ici.

1. **Créez votre compte Azure for Students et votre cluster AKS** (si ce n'est pas déjà fait), en
   suivant les étapes de
   [Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}})
   (création du compte, groupe de ressources, `az aks create`).
2. *(Optionnel, pour pratiquer)* **Créez un namespace** pour votre équipe — pas strictement
   nécessaire puisque vous êtes seuls sur votre cluster, mais utile à pratiquer :

   ```shell
   kubectl create namespace equipe-<votre-numero>
   kubectl config set-context --current --namespace=equipe-<votre-numero>
   ```

3. **Vérifiez l'accès à votre image GHCR** : si le *package* `petclinic` publié à la semaine 7 est
   resté privé, suivez [Azure Kubernetes Service (AKS), section 3️⃣]({{% relref "/semaine-11/azure-kubernetes-service" %}})
   pour le rendre public ou créer le secret `ghcr-secret`.
4. **Écrivez un Deployment** pour l'image `petclinic` que vous avez publiée sur GHCR à la
   semaine 7 (`ghcr.io/<votre-compte>/petclinic:latest`), avec `replicas: 2` et le port `9966`, sur
   le modèle de [Kubernetes fondamentaux, section 3️⃣]({{% relref "/semaine-11/kubernetes-fondamentaux" %}})
   (ajoutez `imagePullSecrets` si vous avez créé `ghcr-secret` à l'étape précédente). Appliquez-le
   avec `kubectl apply -f deployment.yaml`.
5. **Observez le démarrage des pods** en temps réel :

   ```shell
   kubectl get pods -w
   ```

   Attendez que les deux pods passent à `Running`, puis arrêtez l'observation (`Ctrl+C`). Si un
   pod reste en `ImagePullBackOff`, retournez à l'étape 3.
6. **Écrivez un Service** de type `LoadBalancer` pour exposer ce Deployment, sur le modèle de
   [Kubernetes fondamentaux, section 4️⃣]({{% relref "/semaine-11/kubernetes-fondamentaux" %}}).
   Appliquez-le, récupérez l'IP externe (`kubectl get services`), et confirmez l'accès avec :

   ```shell
   curl http://<ip-externe>:9966/petclinic/api/vets
   ```

7. **Testez la répartition de charge et l'auto-guérison** :
   - Augmentez les réplicas à 4 (`kubectl scale deployment/petclinic --replicas=4`) et confirmez
     que le Service répond toujours, sans aucune modification de son propre manifeste.
   - Supprimez manuellement un pod (`kubectl delete pod <nom-du-pod>`) et observez, avec
     `kubectl get pods -w`, qu'un nouveau pod est automatiquement recréé pour revenir à l'état
     désiré.
8. **Arrêtez votre cluster AKS** dès que vous avez terminé votre session de travail, pour préserver
   votre crédit étudiant :

   ```shell
   az aks stop --name <nom-cluster> --resource-group <nom-groupe>
   ```

Pour la Pull Request (dans le dépôt où vous conservez vos manifestes Kubernetes) :

- Les fichiers `deployment.yaml` et `service.yaml` utilisés.
- Une capture de `kubectl get pods` montrant vos réplicas `Running`.
- Une capture ou une description du test d'auto-guérison (pod supprimé manuellement, puis
  recréé automatiquement).
- Une confirmation que vous avez arrêté votre cluster AKS à la fin de votre session de travail.

---

## Pour la prochaine fois

La semaine 12 reprendra ce même Deployment/Service pour y ajouter un **ConfigMap**, un **Secret**,
un **Ingress**, puis les empaqueter dans un **chart Helm**, pour y déployer une mise à jour sans
interruption de service.
