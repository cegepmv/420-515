+++
title = "Lire le code des autres"
weight = 1
+++

## Stratégies de lecture de code

Lire du code écrit par quelqu'un d'autre est une compétence à part entière, distincte de l'écriture
de code. Quelques stratégies complémentaires :

- **Lecture top-down** : partir des points d'entrée du programme (les endpoints d'une API, les
  contrôleurs, l'interface utilisateur) et descendre progressivement vers les détails
  d'implémentation. Utile pour comprendre *ce que fait* le système avant *comment* il le fait.
- **Lecture bottom-up** : partir des entités/du modèle de données et remonter vers les couches
  supérieures. Utile quand on cherche à comprendre une structure de données précise ou une règle
  métier.
- **Utiliser les tests existants comme documentation vivante** : un bon test unitaire montre
  souvent, mieux qu'un commentaire, ce que le code est censé faire — y compris les cas limites.
- **Utiliser le débogueur pour suivre un flux d'exécution réel** plutôt que de deviner en lisant
  seulement le code statique : poser un point d'arrêt et observer l'état réel des variables est
  souvent plus rapide et plus fiable que de « lire dans sa tête ».

### Exemple générique

Prenons une méthode typique d'un contrôleur qui expose une liste de ressources :

```java
@GetMapping("/resources")
public ResponseEntity<List<ResourceDto>> getResources() {
    List<Resource> resources = this.service.findAllResources();
    if (resources.isEmpty()) {
        return new ResponseEntity<>(HttpStatus.NOT_FOUND);
    }
    return new ResponseEntity<>(mapper.toDtoCollection(resources), HttpStatus.OK);
}
```

Questions à se poser en la lisant : d'où vient `service` ? Qu'est-ce qu'un `mapper` et pourquoi
existe-t-il (séparer le modèle interne du format exposé à l'extérieur) ? Est-ce une bonne pratique
de retourner `404` sur une liste vide plutôt qu'une liste vide avec `200` ? (Il n'y a pas de
consensus universel ici — c'est une discussion de design d'API qu'on approfondira plus tard dans la
session, lorsqu'on abordera l'évolution des API.)

![Stratégies de lecture de code](images/lecture-code-strategies.svg)

---

## Premier repérage guidé de code smells

Sans encore utiliser le vocabulaire officiel (il sera présenté en détail plus tard dans la
session), voici une grille de lecture simple à utiliser en équipe sur quelques classes du projet :

| Question à se poser en lisant | Smell potentiel |
|---|---|
| Cette méthode fait-elle plus de 20-30 lignes ? | *Long Method* |
| Est-ce que je vois le même bloc de code ailleurs dans le projet ? | *Duplicate Code* |
| Cette méthode a-t-elle plus de 4 paramètres ? | *Long Parameter List* |
| Est-ce que je dois ouvrir 3-4 fichiers différents pour comprendre UNE fonctionnalité ? | *Shotgun Surgery* |
| Le nom de la méthode/variable dit-il vraiment ce qu'elle fait ? | Mauvais nommage (base de tout refactoring) |

Remplissez cette grille dans votre `NOTES.md` — ce sera la matière première réutilisée plus tard
dans la session lors de l'introduction formelle du catalogue des code smells et du réusinage.
