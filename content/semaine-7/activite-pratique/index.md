+++
title = "Activité pratique"
weight = 5
draft = false
+++

## 🎯 Activité pratique (non notée)

En équipe, réalisez le travail suivant sur votre dépôt, via une vraie Pull Request, en réinvestissant
directement la démarche vue dans [Docker]({{% relref "/semaine-7/docker" %}}) et
[Registres de conteneurs]({{% relref "/semaine-7/registre-conteneurs" %}}) :

1. **Écrivez un `Dockerfile` en construction multi-étapes** pour votre projet (backend), en
   appliquant les bonnes pratiques vues cette semaine (image JRE pour l'exécution, utilisateur
   non-root, `.dockerignore`).
2. **Validez-le localement** : construisez l'image, démarrez un conteneur, confirmez son bon
   fonctionnement avec une requête réelle.
3. **Ajoutez un job de publication** à votre pipeline CI (`ci.yml`), qui construit et
   publie automatiquement l'image sur GitHub Container Registry après chaque fusion sur `main`,
   seulement si la suite de tests a réussi.
4. **Confirmez**, par une capture ou un lien vers l'onglet **Packages** de votre dépôt, que l'image
   est bien publiée et accessible.

Pour la Pull Request :

- Le `Dockerfile` et le fichier `.dockerignore` ajoutés.
- Le job de publication ajouté à votre workflow CI existant.
- Une description claire de la stratégie de tags utilisée (SHA du commit + `latest`, et pourquoi).
