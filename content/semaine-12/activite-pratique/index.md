+++
title = "Activité pratique"
weight = 4
draft = false
+++

## 🎯 Déploiement via Helm, avec rolling update

En équipe, en réinvestissant directement
[ConfigMap, Secrets, logs, Ingress]({{% relref "/semaine-12/configuration-kubernetes" %}}) et
[Helm et déploiement sans interruption]({{% relref "/semaine-12/helm-deploiement-continu" %}}), à
partir du Deployment/Service `petclinic` déjà déployé à l'
[activité pratique de la semaine 11]({{% relref "/semaine-11/activite-pratique" %}}) :

> ⚠️ **C'est une remise à soumettre.** Réalisable sur votre cluster **AKS**
> (voir [Azure Kubernetes Service (AKS)]({{% relref "/semaine-11/azure-kubernetes-service" %}})).
> Pensez à arrêter votre cluster AKS (`az aks stop`) entre vos sessions de travail pour préserver
> votre crédit étudiant.

### Étapes

1. **Construisez un chart Helm minimal** pour `petclinic`, à partir de votre Deployment et Service
   de la semaine 11 :
   - Soit en partant d'un squelette généré (`helm create petclinic`, puis en remplaçant le contenu
     des templates par défaut par vos propres manifestes), soit en écrivant manuellement la
     structure `Chart.yaml` / `values.yaml` / `templates/` décrite dans
     [Helm et déploiement sans interruption, section 1️⃣]({{% relref "/semaine-12/helm-deploiement-continu" %}}).
   - Le `values.yaml` doit au minimum paramétrer `image.repository`, `image.tag`, et
     `replicaCount`.

2. **Externalisez au moins un paramètre de configuration** via un `ConfigMap` intégré au chart
   (ex. le profil Spring actif ou l'URL de la base de données), et **au moins une donnée
   sensible** via un `Secret` (ex. le mot de passe de la base de données), sur le modèle de
   [ConfigMap, Secrets, logs, Ingress]({{% relref "/semaine-12/configuration-kubernetes" %}}).
   Montez les deux dans le conteneur `petclinic` du template de Deployment (`envFrom`/`env`).

3. **Déployez une première fois** avec `helm install` :

   ```shell
   helm install petclinic ./petclinic -n equipe-<votre-numero>
   helm status petclinic -n equipe-<votre-numero>
   ```

   Confirmez l'accès à l'application (`curl http://<ip-externe>:9966/petclinic/api/vets`).

4. **Simulez une mise à jour de version** : construisez (ou réutilisez) un nouveau tag d'image
   produit par votre pipeline CI/CD des semaines précédentes (ex. `${{ github.sha }}` ou un
   nouveau tag sémantique — semaine 8), puis appliquez-le avec :

   ```shell
   helm upgrade petclinic ./petclinic --set image.tag=<nouveau-tag> --atomic -n equipe-<votre-numero>
   ```

   (ou `--rollback-on-failure` si votre équipe utilise Helm 4 — vérifiez avec `helm version`).

   Pendant la mise à jour, **observez qu'il n'y a pas d'interruption de service** :
   - Dans un terminal séparé, lancez une boucle de requêtes continues pendant l'upgrade :

     ```shell
     while true; do curl -s -o /dev/null -w "%{http_code}\n" http://<ip-externe>:9966/petclinic/api/vets; sleep 1; done
     ```

   - Dans un autre terminal, suivez le rollout :

     ```shell
     kubectl rollout status deployment/petclinic -n equipe-<votre-numero>
     ```

   - Confirmez qu'aucune requête de la boucle n'a échoué (que des `200`) pendant toute la durée du
     rollout.

5. **(Optionnel/bonus) Exposez l'application via un `Ingress`**, sur le modèle de
   [ConfigMap, Secrets, logs, Ingress, section 4️⃣]({{% relref "/semaine-12/configuration-kubernetes" %}}),
   intégré lui aussi au chart.

### Pour la Pull Request

- Le chart Helm complet (`Chart.yaml`, `values.yaml`, `templates/`), ajouté à votre dépôt.
- Une description de ce qui a été externalisé en `ConfigMap` et en `Secret`, et pourquoi chacun a
  été placé dans l'un plutôt que l'autre.
- Une preuve (capture d'écran ou sortie de terminal) montrant :
  - le résultat de `helm upgrade` avec le nouveau tag,
  - la sortie de `kubectl rollout status` confirmant un rollout réussi,
  - la boucle de requêtes continues, démontrant qu'aucune requête n'a échoué pendant la mise à
    jour.
- La confirmation que vous avez arrêté votre cluster AKS (`az aks stop`) à la fin de votre dernière
  session de travail.
