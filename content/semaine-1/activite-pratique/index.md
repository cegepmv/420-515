+++
title = "Activité pratique et récapitulatif"
weight = 5
+++

## 🎯 Activité pratique

1. **Installez et vérifiez** votre environnement (JDK requis, IntelliJ ou l'éditeur de votre choix).
2. Chaque équipe **clone** le projet du cours, le fait rouler localement, et confirme qu'au moins
   un endpoint de l'API répond bien en JSON.
3. Explorez le code pendant **30-40 minutes**, en équipe, à la recherche de « code smells » — des
   indices que quelque chose semble anormal ou mal conçu : noms de variables cryptiques (`e`, `tmp`,
   `x1`), méthodes qui font plus de 20-30 lignes, blocs de code qui se ressemblent étrangement
   d'un fichier à l'autre (duplication), commentaires qui ne correspondent plus au code en dessous...
4. **Ne corrigez rien encore** — l'objectif aujourd'hui est seulement d'observer et de discuter en
   équipe de vos trouvailles (le nom du fichier concerné et une courte explication pour chacune).
5. On **ne nomme pas encore** les smells avec le vocabulaire officiel de Fowler (*Long Method*,
   *Duplicated Code*, etc.) — ce vocabulaire sera introduit formellement en **semaine 5**.
   Aujourd'hui, l'objectif est seulement d'entraîner votre œil à repérer que « quelque chose sent
   mauvais » ici et là, même sans savoir encore comment ça s'appelle.

<details>
<summary>💬 Question de réflexion à poser en fin de séance</summary>

« Combien d'entre vous ont déjà écrit, sans le savoir, un des smells que vous venez de repérer dans
du code d'un autre ? »

(Presque toute la classe lève la main habituellement — excellent moment pour désamorcer le
jugement et introduire l'idée que le refactoring n'est pas une critique personnelle envers celui ou
celle qui a écrit le code, c'est un entretien normal et attendu du code, un peu comme changer
l'huile d'une voiture : ce n'est pas parce que le mécanicien précédent a mal fait son travail, c'est
simplement l'usure normale du temps et des changements de contexte.)
</details>

---

## ✅ Auto-vérification avant la semaine 2

Avant la prochaine séance, assurez-vous d'être capable de répondre "oui" à chacune de ces questions :

- [ ] Mon JDK est dans la version exigée par le projet (`java -version` confirmé).
- [ ] J'ai réussi à cloner le projet et à le démarrer localement sans erreur.
- [ ] J'ai testé au moins un endpoint de l'API et obtenu une réponse JSON.
- [ ] Je sais expliquer, dans mes mots, la différence entre maintenance corrective, adaptative,
      perfective et préventive.
- [ ] J'ai identifié, en équipe, au moins 2-3 endroits du code qui me semblent « louches », même
      sans savoir pourquoi exactement.
