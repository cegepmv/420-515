+++
title = "Activité pratique"
weight = 8
+++

## 🎯 Activité pratique (non notée)

En équipe, réalisez les **deux parties** suivantes, chacune via une vraie Pull Request sur GitHub :

- **Partie A** : corriger un premier bogue simple dans votre projet.
- **Partie B** : ajouter une toute petite fonctionnalité à un endpoint existant.

Pour chaque Pull Request :

- Une description claire de ce qui a été fait (bogue corrigé, ou fonctionnalité ajoutée).
- Au moins une revue de code croisée entre équipes avant la fusion.

### 🐞 Partie A — Corriger un bogue

#### Où trouver ce bogue ?

Cette page ne dicte pas *quel* bogue corriger, puisque chaque équipe travaille sur son propre
projet — mais voici, dans l'ordre à essayer, les sources les plus courantes :

1. **La documentation/le README fourni avec votre projet** — certains projets legacy incluent
   volontairement des indices sur des bogues à découvrir (souvent une section du genre « pistes de
   réflexion » ou « bogues connus »), justement pour vous entraîner à les trouver vous-mêmes sans
   tout vous révéler d'un coup. Vérifiez d'abord si votre projet en a une.
2. **La liste de bogues connus fournie par l'enseignant** pour votre projet (si elle existe
   séparément du README) — un autre point de départ simple.
3. **Les *issues* GitHub ouvertes** sur le dépôt du projet, si le projet en contient déjà.
4. **Un bogue que vous avez vous-même repéré** durant l'activité de
   [lecture de code]({{< relref "/semaine-2/lecture-de-code" >}}) de cette semaine — par exemple
   via la grille de code smells (nommage trompeur, méthode qui fait clairement autre chose que ce
   que son nom indique) ou une incohérence remarquée entre deux couches.
5. **Un bogue trouvé en explorant l'application** vous-même : un cas limite non géré (valeur nulle,
   liste vide, entrée invalide) qui produit une erreur ou un comportement clairement incorrect.

> ⚠️ Choisissez un bogue **simple et isolé** — une seule méthode ou un seul comportement précis à
> corriger. L'objectif ici est de pratiquer le cycle complet (branche → commit → PR → revue), pas
> de résoudre le bogue le plus complexe du projet.

### 🆕 Partie B — Ajouter une petite fonctionnalité à un endpoint existant

Contrairement à la Partie A (corriger quelque chose qui ne fonctionne pas), il s'agit ici
d'**étendre légèrement** un endpoint qui fonctionne déjà correctement — ce qui vous oblige à
d'abord bien comprendre son fonctionnement actuel avant d'y toucher (mise en pratique directe de la
[lecture top-down et bottom-up]({{< relref "/semaine-2/lecture-de-code" >}}) vue cette semaine).

**Démarche suggérée** :
1. Choisissez un endpoint existant simple (ex. celui qui retourne une liste d'éléments, ou un
   élément par identifiant).
2. Repérez son point d'entrée (contrôleur), puis descendez vers le service et l'accès aux données
   pour bien comprendre le chemin complet que suit une requête.
3. Ajoutez la petite fonctionnalité **sans changer le comportement existant** (les appels actuels
   à cet endpoint doivent continuer de fonctionner exactement comme avant).

**Exemples génériques de petites fonctionnalités à ajouter** (adaptez à votre projet) :

- Un **paramètre de filtrage optionnel** sur un endpoint qui retourne une liste (ex. filtrer par
  un champ existant, en ne renvoyant que les éléments qui correspondent).
- Un **champ calculé** ajouté à la réponse existante (ex. un total, un compte, une valeur dérivée
  d'autres champs déjà présents), sans devoir changer le modèle de données en base.
- Un **paramètre de tri optionnel** (ex. trier par un champ précis, croissant ou décroissant).
- Une **petite validation supplémentaire** sur un endpoint de création/modification (ex. rejeter
  une valeur clairement invalide qui n'était pas vérifiée avant).

> 💡 Le but n'est pas la complexité, mais de **pratiquer la recherche du bon endroit à modifier**
> dans du code que vous n'avez pas écrit — le même réflexe que vous utiliserez chaque fois qu'on
> vous demandera une évolution sur un projet existant, en cours comme en emploi.

Cette activité met immédiatement en pratique le cycle complet vu cette semaine : lecture de code,
création d'une branche, commit, push, ouverture d'une Pull Request, et revue de code.

> 📌 Cette activité **prépare** le TP1 noté (10 points), prévu à la **semaine 3** selon le plan de
> cours — ce n'est pas la remise elle-même. La semaine 2 est plutôt évaluée par le **Quiz 1**
> (3 points).
