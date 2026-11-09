+++
title = "Activité pratique"
weight = 4
draft = false
+++

## 🎯 Activité pratique

En équipe, sur votre application legacy, en réinvestissant directement
[Analyse statique avec SonarQube / SonarCloud]({{% relref "/semaine-9/analyse-statique-sonarqube" %}})
et [Analyse de vulnérabilités avec Trivy]({{% relref "/semaine-9/analyse-vulnerabilites-trivy" %}}) :

1. **Créez un projet SonarCloud** lié à votre dépôt GitHub (compte gratuit pour un dépôt public),
   récupérez votre `SONAR_TOKEN`, et ajoutez-le comme secret de votre dépôt.
2. **Ajoutez une étape d'analyse SonarCloud** à votre pipeline GitHub Actions existant (`ci.yml`,
   semaine 5), après l'étape qui exécute déjà vos tests — sur le modèle de la commande Maven vue
   dans les notes (`sonar-maven-plugin:sonar`, avec vos propres `sonar.projectKey` et
   `sonar.organization`).
3. **Observez le quality gate** de votre premier commit analysé : s'il est `Not computed`, relancez
   l'analyse sur un second commit pour voir apparaître un vrai statut `Passed`/`Failed`.
4. **Provoquez volontairement un échec du quality gate** pour observer le pipeline bloquer un
   merge, avant de corriger quoi que ce soit :
   - Ajoutez `-Dsonar.qualitygate.wait=true` à votre commande `sonar-maven-plugin:sonar` (sans ce
     paramètre, l'analyse est asynchrone et ne fait jamais échouer le build Maven, même si le
     quality gate échoue — voir
     [Analyse statique, section 7️⃣]({{% relref "/semaine-9/analyse-statique-sonarqube" %}})).
   - Sur une branche dédiée, introduisez une vulnérabilité **Blocker** facile à provoquer : une
     requête construite par concaténation de chaînes passée telle quelle à `createQuery(...)`. Ce
     patron déclenche la règle d'injection SQL/JPQL (sévérité Blocker, type Vulnerability), ce qui
     fait chuter la Security Rating du **nouveau code** sous le seuil requis par le quality gate
     par défaut.

     Fichier à modifier :
     `src/main/java/org/springframework/samples/petclinic/repository/jpa/JpaPetRepositoryImpl.java`

     Ajoutez cette méthode dans la classe `JpaPetRepositoryImpl` (par exemple juste avant la
     méthode `delete`) :

     ```java
     /**
      * DEMO: méthode volontairement vulnérable pour illustrer l'échec du quality gate
      * SonarCloud (injection SQL/JPQL via concaténation de chaînes - règle S3649, sévérité Blocker).
      * À retirer après la démonstration.
      */
     @SuppressWarnings("unchecked")
     public List<Pet> findByNameUnsafe(String name) {
         String jpql = "SELECT pet FROM Pet pet WHERE pet.name = '" + name + "'";
         return this.em.createQuery(jpql).getResultList();
     }
     ```

   - Ouvrez une Pull Request avec ce changement vers votre branche principale : confirmez que le
     job d'analyse échoue dans GitHub Actions et que la PR affiche le merge comme bloqué.
   - Retirez ensuite ce code de démonstration (ou corrigez-le avec une requête paramétrée) avant de
     poursuivre.
5. **Corrigez au moins 3 *issues*** signalées par SonarCloud sur votre code (pas sur du code
   généré) — en priorisant les plus sévères, et en documentant, pour chacune, la qualité logicielle
   concernée (sécurité, fiabilité, ou maintenabilité) et la règle enfreinte.
6. **Ajoutez une étape Trivy** à votre pipeline pour scanner l'image Docker construite à la
   semaine 7, avec `severity: 'CRITICAL,HIGH'` et `exit-code: '1'`.
7. **Examinez le rapport Trivy** : si des vulnérabilités `CRITICAL`/`HIGH` corrigibles sont
   trouvées (souvent dans l'image de base ou une dépendance obsolète), mettez à jour la version
   fautive et confirmez que le scan repasse au vert.

Pour la Pull Request :

- Les étapes SonarCloud et Trivy ajoutées à `ci.yml`, avec les secrets correctement configurés.
- Une capture ou un lien du quality gate SonarCloud de votre projet.
- La liste des *issues* corrigées (qualité logicielle, règle, description du correctif).
- La confirmation que le scan Trivy passe (ou la justification documentée d'une vulnérabilité
  volontairement ignorée, par exemple via `ignore-unfixed`).