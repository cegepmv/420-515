+++
title = "Exercice : qualité et sécurité du code"
weight = 3
draft = false
+++

## 🎯 Objectif

Appliquer les concepts vus dans
[Analyse statique avec SonarQube / SonarCloud]({{% relref "/semaine-9/analyse-statique-sonarqube" %}})
et [Analyse de vulnérabilités avec Trivy]({{% relref "/semaine-9/analyse-vulnerabilites-trivy" %}})
à des situations concrètes de pipeline CI/CD.

---

## 🧩 Partie 1 — Classer des *issues* SonarQube

Pour chacune des situations suivantes, indiquez la **qualité logicielle** principalement concernée
(sécurité, fiabilité, ou maintenabilité — voir
[Analyse statique]({{% relref "/semaine-9/analyse-statique-sonarqube" %}})), et justifiez votre
réponse à l'aide des définitions officielles vues dans les notes :

1. Une méthode construit une requête SQL par concaténation directe de chaînes de caractères avec
   une entrée fournie par l'utilisateur.
2. Une méthode contient 40 lignes de code dupliquées, presque identiques, à trois endroits
   différents du projet.
3. Une méthode effectue une division par une variable sans jamais vérifier qu'elle est différente
   de zéro.
4. Un bloc de code gère une connexion à une ressource externe (fichier, flux réseau) sans jamais
   la refermer, même en cas d'exception.

---

## 🧩 Partie 2 — Lire un pipeline CI/CD réel

En vous basant sur le workflow `maven-build-master.yml` du projet `spring-petclinic-rest` (voir
[Analyse statique]({{% relref "/semaine-9/analyse-statique-sonarqube" %}})) :

```yaml
- name: Build and analyze
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: ./mvnw -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=spring-petclinic_spring-petclinic-rest -Dsonar.organization=spring-petclinic
```

1. Pourquoi l'analyse SonarCloud est-elle invoquée **après** `verify` plutôt qu'en remplacement de
   celui-ci ? Que perdrait-on si l'équipe exécutait uniquement
   `./mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:sonar`, sans `verify` ?
2. Pourquoi ce workflow a-t-il besoin d'un `SONAR_TOKEN` en plus du `GITHUB_TOKEN` déjà utilisé
   dans les workflows vus à la semaine 5 ?
3. Le `pom.xml` du projet configure JaCoCo avec un minimum de 85 % de couverture de lignes
   (`LINE`, `COVEREDRATIO`, `0.85`). Si une Pull Request fait chuter la couverture du **nouveau
   code** à 60 %, mais que la couverture **globale** du projet reste au-dessus de 85 % grâce au
   code déjà existant, le build Maven (`jacoco:check`) échoue-t-il nécessairement ? Justifiez en
   distinguant la portée de la règle JaCoCo (`BUNDLE`, donc globale) de celle d'un quality gate
   SonarCloud (qui cible spécifiquement le « nouveau code », voir
   [Analyse statique, section 4️⃣]({{% relref "/semaine-9/analyse-statique-sonarqube" %}})).

---

## 🧩 Partie 3 — Configurer un scan Trivy

Vous devez ajouter une étape Trivy à un pipeline qui construit déjà une image Docker (semaine 7).
Écrivez les options `with:` nécessaires (sur le modèle vu dans
[Analyse de vulnérabilités]({{% relref "/semaine-9/analyse-vulnerabilites-trivy" %}})) pour
répondre à **chacune** des exigences suivantes, séparément :

1. Le pipeline doit échouer si une vulnérabilité `CRITICAL` **ou** `HIGH` est trouvée.
2. Le pipeline ne doit **pas** échouer à cause d'une vulnérabilité pour laquelle aucun correctif
   n'est encore disponible.
3. Les résultats doivent être consultables dans l'onglet **Security** du dépôt GitHub, même si le
   scan fait échouer le pipeline.
4. Le fichier `trivy-results.sarif` brut doit rester consultable après coup (ex. pour déboguer un
   problème de format), même si l'onglet Security n'affiche que les alertes déjà interprétées.

Pour l'exigence 3, précisez également la permission GitHub Actions requise, en plus des options
`with:` de l'action Trivy elle-même.

---

<details>
<summary>❓ Corrigé — Partie 1</summary>

1. **Sécurité** — c'est une injection SQL classique : le code pourrait être exploité par un
   attaquant qui contrôle l'entrée utilisateur pour altérer la requête exécutée.
2. **Maintenabilité** — la duplication n'est ni un bug ni une vulnérabilité; elle rend le code plus
   difficile à comprendre et à faire évoluer (une correction doit être répétée à trois endroits).
3. **Fiabilité** — une division par zéro est un code « manifestement erroné, ou plus probablement
   erroné que non » (définition officielle d'une règle de fiabilité) : elle provoquera une
   exception à l'exécution dans certains cas.
4. **Fiabilité** — une ressource non refermée est un défaut classique (fuite de ressource) qui finit
   par causer un comportement incorrect de l'application (épuisement des connexions, par exemple),
   même si ce n'est pas, en soi, exploitable par un attaquant.
</details>

<details>
<summary>❓ Corrigé — Partie 2</summary>

1. `verify` déclenche tout le cycle de vie Maven jusque-là (compilation, tests, empaquetage —
   semaine 3) **et**, grâce à `jacoco:prepare-agent`, instrumente le code pour produire le rapport
   de couverture consommé par SonarCloud. Sans `verify`, les tests ne seraient pas exécutés dans
   cette invocation et SonarCloud analyserait le code source sans aucune donnée de couverture
   récente, faussant les métriques de qualité liées aux tests.
2. Le `GITHUB_TOKEN` est généré automatiquement par GitHub pour interagir avec l'API GitHub (ex.
   décorer une Pull Request). Le `SONAR_TOKEN` est un jeton **distinct**, généré depuis SonarCloud,
   qui authentifie le pipeline **auprès de SonarCloud** — un service tiers qui n'a aucun lien de
   confiance automatique avec GitHub.
3. Non, pas nécessairement. La règle JaCoCo configurée dans le `pom.xml` (`element: BUNDLE`) évalue
   la couverture de **l'ensemble** du module, pas seulement du code modifié par la PR — tant que la
   moyenne globale reste ≥ 85 %, `jacoco:check` passe, peu importe la couverture du nouveau code
   pris isolément. C'est le quality gate SonarCloud, configuré séparément avec une condition sur le
   « New Code », qui peut détecter ce problème et faire échouer la PR malgré un build Maven vert.
</details>

<details>
<summary>❓ Corrigé — Partie 3</summary>

1. `severity: 'CRITICAL,HIGH'` et `exit-code: '1'`.
2. `ignore-unfixed: true`.
3. `format: 'sarif'`, `output: 'trivy-results.sarif'`, puis une étape
   `github/codeql-action/upload-sarif@v4` avec `sarif_file: 'trivy-results.sarif'` et
   `if: always()` (pour téléverser même en cas d'échec). Permission requise au niveau du job :
   `permissions: security-events: write` (en plus de `contents: read`).
4. Une étape `actions/upload-artifact@v4` avec `path: trivy-results.sarif` et `if: always()`. Sans
   cette étape, le fichier brut n'est jamais conservé : il est produit sur le système de fichiers
   éphémère du runner, lu par `upload-sarif`, puis disparaît avec le runner à la fin du job — seul
   le contenu interprété (les alertes) reste visible dans l'onglet Security.
</details>

---

## ✅ Auto-vérification

- [ ] Je peux expliquer, avec mes mots, la différence entre les qualités **sécurité**, **fiabilité**
      et **maintenabilité** de SonarQube, et donner un exemple de règle pour chacune.
- [ ] Je peux expliquer pourquoi un quality gate peut échouer sur une Pull Request même si la
      couverture globale du projet reste élevée.
- [ ] Je peux écrire, sans copier un exemple, les options Trivy nécessaires pour faire échouer un
      pipeline uniquement sur des vulnérabilités critiques/élevées et corrigibles.
- [ ] Je sais quelle permission GitHub Actions est requise pour publier un rapport SARIF dans
      l'onglet Security.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[activité pratique]({{% relref "/semaine-9/activite-pratique" %}}) pour intégrer
réellement SonarCloud et Trivy à votre propre pipeline, et corriger ce qu'ils y trouvent.
{{% /notice %}}
