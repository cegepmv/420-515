+++
title = "Semaine 10 — Distribution de mises à jour et gestion des bogues"
type = "chapter"
weight = 11
draft = false
+++

## 🎯 Objectifs d'apprentissage de la semaine

À la fin de cette semaine, vous devriez être capable de :

- Expliquer pourquoi **distribuer** une mise à jour est un problème distinct de celui de la
  **construire** (semaines 5 à 9) : une fois le pipeline vert et l'image publiée, encore faut-il
  qu'elle atteigne chaque appareil en production, de façon fiable et sécurisée.
- Décrire les contraintes particulières de mise à jour des **objets connectés** (bande passante,
  disponibilité intermittente, impossibilité de revenir physiquement sur l'appareil) et pourquoi
  l'absence de mise à jour y est un risque de sécurité documenté.
- Expliquer ce qu'est le **rétroportage de correctifs** (*backporting*) : pourquoi une équipe
  applique parfois un correctif à une ancienne version plutôt que d'imposer la dernière version à
  tous ses utilisateurs.
- Expliquer pourquoi une mise à jour doit être **signée numériquement**, et ce que cette signature
  garantit (et ne garantit pas) face à un attaquant qui intercepterait ou altérerait le paquet.
- Créer, étiqueter et lier des **GitHub Issues** à des Pull Requests, dans une optique de suivi de
  bogues outillé plutôt qu'informel.

---

## 🔍 Accroche

« Les neuf premières semaines de ce cours ont construit un pipeline complet : build, tests,
réusinage, conteneurisation, versionnage, qualité et sécurité du code. Supposons que tout ce
pipeline tourne parfaitement et qu'une image corrigée vient d'être publiée dans le registre de
conteneurs (semaine 7). Un appareil IoT déployé chez un client depuis deux ans, actuellement
derrière un pare-feu, sans écran et sans clavier, reçoit-il cette mise à jour ? Comment ? Et
comment ce même appareil peut-il être certain que le fichier qu'il vient de recevoir provient
réellement de vous, et non d'un attaquant qui aurait intercepté la connexion ? Cette semaine
s'intéresse à ce qui se passe **après** la publication — et au dernier maillon encore manquant du
cycle de maintenance : le suivi outillé des bogues eux-mêmes. »
