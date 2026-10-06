+++
title = "Semaine 7 — Conteneurisation avec Docker"
type = "chapter"
weight = 8
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Expliquer la différence entre une image et un conteneur, et pourquoi la conteneurisation résout
  le problème classique du « ça fonctionne sur ma machine ».
- Écrire un `Dockerfile` pour une application Spring Boot, en appliquant les bonnes pratiques de
  base (image de base minimale, construction multi-étapes, utilisateur non-root, ordre des
  instructions qui maximise la réutilisation du cache).
- Construire une image Docker localement, démarrer un conteneur à partir de cette image, et
  valider son bon fonctionnement avec de vraies requêtes HTTP.
- Expliquer ce qu'est un registre de conteneurs (*container registry*) et distinguer un registre
  public généraliste (Docker Hub) d'un registre lié à votre dépôt de code (GitHub Container
  Registry).
- Intégrer la construction et la publication automatisée d'une image Docker dans un pipeline
  GitHub Actions existant (semaine 5), vers GitHub Container Registry.
- Choisir une stratégie de nommage des étiquettes (*tags*) d'image cohérente (ex. le SHA du
  commit, `latest`) pour retrouver facilement une version précise de l'image publiée.

---

## 🔍 Accroche

« Votre pipeline CI (semaine 5) prouve que le code compile et que les tests passent — mais sur
quelle machine, exactement, ce code va-t-il *tourner* une fois livré ? La vôtre ? Celle d'un
collègue ? Un serveur qui n'a pas exactement la même version du JDK, ni les mêmes bibliothèques
système ? Docker répond à cette question une fois pour toutes : on n'emballe plus seulement le
code, on emballe **l'environnement complet** dans lequel il s'exécute, de façon identique partout —
de votre poste jusqu'au serveur de production. »
