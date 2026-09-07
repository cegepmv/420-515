+++
title = "Techniques de débogage (2/2)"
weight = 1
+++

En semaine 2, vous avez appris la **méthode** : reproduire, isoler, lire la stack trace, formuler
une hypothèse. Cette semaine ajoute les **outils** qui rendent chaque étape rapide : le débogueur
pas à pas, les breakpoints conditionnels, une journalisation propre, et la bissection pour localiser
une cause sans tout lire ligne par ligne.

---

## 1️⃣ Le débogueur pas à pas

Un **breakpoint** (point d'arrêt) suspend l'exécution à une ligne précise; vous inspectez alors
l'état réel du programme (variables, pile d'appels) au lieu de le deviner. La documentation
officielle d'IntelliJ IDEA résume la démarche générale en 3 étapes, peu importe l'IDE utilisé :

1. **Définir où le programme doit s'arrêter** — un ou plusieurs breakpoints.
2. **Lancer le programme en mode Debug** (application, test unitaire, ou tout code exécutable).
3. **Examiner l'état du programme suspendu**, puis avancer pas à pas (*stepping*) pour observer
   comment cet état évolue à chaque ligne.

| Action | IntelliJ IDEA | VS Code |
|---|---|---|
| Poser/retirer un breakpoint | Clic dans la marge, ou `Ctrl+F8` | Clic dans la marge, ou `F9` |
| Démarrer en mode Debug | `Shift+F9` | `F5` |
| *Step Over* (ligne suivante, sans entrer dans les méthodes appelées) | `F8` | `F10` |
| *Step Into* (entrer dans la méthode appelée) | `F7` | `F11` |
| *Step Out* (sortir de la méthode courante) | `Shift+F8` | `Shift+F11` |
| Reprendre l'exécution normale | `F9` | `F5` |

**Exemple.** Un calcul de total sur une liste retourne une valeur inattendue :

```java
public double calculerTotal(List<Item> items) {
    double total = 0;
    for (Item item : items) {
        total += item.getPrix() * item.getQuantite();
    }
    return total;
}
```

Démarche : posez un breakpoint sur la ligne `total += ...`, démarrez en Debug, puis *Step Over* à
chaque itération en observant `total`, `item.getPrix()` et `item.getQuantite()` dans le panneau
**Variables**. Vous verrez précisément à **quelle itération** la valeur dévie de ce qui est attendu.

> 💡 **Watches et Evaluate Expression** — deux fonctionnalités à connaître au-delà du panneau
> Variables : un **Watch** garde une expression précise sous surveillance en permanence pendant
> toute la session (ex. `total / items.size()`, même si ce calcul n'existe pas dans le code) ;
> **Evaluate Expression** (`Alt+F8`) exécute une expression arbitraire *pendant* la pause, sans
> modifier le code — utile pour tester une hypothèse (« est-ce que `item.getPrix() < 0` est vrai
> ici ? ») sans ajouter puis retirer une ligne de code.

<details>
<summary>🤔 Testez-vous</summary>


Le total final est plus élevé que prévu, mais seulement légèrement. Utiliseriez-vous *Step Over*
ou *Step Into* sur la ligne `total += item.getPrix() * item.getQuantite();` ? Pourquoi ?

**Réponse** : *Step Over* d'abord — le bogue est probablement dans les **données** (un prix ou une
quantité incorrecte à une itération précise), pas dans la logique interne de `getPrix()`/
`getQuantite()`. *Step Into* ne serait utile que si vous suspectiez un calcul erroné **à l'intérieur**
de l'un de ces getters (ex. une conversion de devise cachée).
</details>

---

## 2️⃣ Breakpoints conditionnels

Poser un breakpoint classique sur une boucle de 500 éléments oblige à cliquer « continuer » des
centaines de fois avant d'atteindre le cas qui vous intéresse. Un **breakpoint conditionnel** ne
s'arrête que lorsqu'une expression booléenne est vraie.

**Comment faire** (IntelliJ et VS Code) : clic droit sur un breakpoint existant → un champ
« Condition » apparaît → entrez une expression Java valide à cet endroit du code.

**Exemple** : dans la boucle ci-dessus, si vous soupçonnez que l'item avec `id == 42` est en cause :

```
Condition : item.getId() == 42
```

Le programme s'exécute normalement pour les 41 premiers éléments, puis s'arrête **exactement** sur
celui qui vous intéresse.

> 💡 Autre usage fréquent : arrêter seulement quand une valeur devient `null` ou négative
> (`item.getQuantite() < 0`) — utile pour attraper un cas limite rare, sans savoir à l'avance à
> quelle itération il se produit.

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Trouvez une boucle ou une collection dans votre projet (ex. une liste retournée par un endpoint).
Posez un breakpoint conditionnel qui ne s'arrête que sur un élément précis (par identifiant, ou par
une valeur inhabituelle d'un champ). Combien de clics « continuer » avez-vous évités par rapport à
un breakpoint classique ?
{{% /notice %}}

---

## 3️⃣ Journalisation stratégique : arrêter d'utiliser `println`

`System.out.println("ici")` est tentant, mais pose 3 problèmes en production :

1. **Aucun niveau de gravité** — impossible de l'activer/désactiver sans modifier le code.
2. **Aucun contexte automatique** — pas d'horodatage, pas de nom de classe, pas de thread.
3. **On oublie de le retirer** — il pollue les logs de production, ou pire, révèle des données
   sensibles.

Un **framework de journalisation** (SLF4J + Logback en Java, le standard de l'industrie) règle ces
trois problèmes avec des **niveaux** :

| Niveau | Usage typique |
|---|---|
| `TRACE` | Détail extrême, désactivé presque toujours (ex. contenu complet d'une requête) |
| `DEBUG` | Informations utiles seulement pendant le développement/débogage |
| `INFO` | Événements normaux notables (ex. « commande #123 créée ») |
| `WARN` | Quelque chose d'anormal mais non bloquant (ex. valeur par défaut utilisée) |
| `ERROR` | Une opération a échoué et nécessite attention |

```java
private static final Logger logger = LoggerFactory.getLogger(FactureService.class);

public double calculerTotal(List<Item> items) {
    double total = 0;
    for (Item item : items) {
        logger.debug("Item {} : prix={}, quantite={}", item.getId(), item.getPrix(), item.getQuantite());
        total += item.getPrix() * item.getQuantite();
    }
    logger.info("Total calculé : {}", total);
    return total;
}
```

- Le `{}` évite la concaténation de chaînes coûteuse quand le niveau `DEBUG` est désactivé.
- En production, on configure typiquement `INFO` ou `WARN` — les lignes `debug()` restent dans le
  code, prêtes à être réactivées sans redéployer, mais n'encombrent pas les logs normalement.

<details>
<summary>🤔 Testez-vous</summary>

Pourquoi `logger.debug(...)` est-il préférable à `System.out.println(...)` même si, au final, les
deux affichent une ligne dans la console pendant que vous déboguez localement ?

**Réponse** : `logger.debug` peut être **désactivé globalement** en production sans toucher au
code (juste la configuration), inclut automatiquement classe/horodatage, et peut être redirigé
(fichier, système centralisé) — `println` ne peut faire aucune de ces trois choses.
</details>

---

## 4️⃣ Bissection : diviser pour localiser

Quand un bogue est « quelque part dans ces 300 lignes » ou « apparu à un moment donné dans les 40
derniers commits », lire séquentiellement est lent. La **bissection** (recherche binaire) élimine
la moitié des suspects à chaque test.

![Bissection : diviser pour localiser la cause](images/bissection.svg)

### Bissection dans le code

Commentez/désactivez la **moitié** du code suspect, relancez le scénario reproductible (semaine 2) :
- Le bogue disparaît → la cause est dans la moitié désactivée.
- Le bogue persiste → la cause est dans l'autre moitié.

Répétez sur la moitié restante jusqu'à isoler une seule ligne ou méthode.

### Bissection dans l'historique Git — `git bisect`

Quand un bogue est apparu « à un moment donné » sans qu'on sache dans quel commit, `git bisect`
automatise exactement la même logique sur l'historique :

```bash
git bisect start
git bisect bad                # le commit actuel (HEAD) est mauvais
git bisect good v1.2.0         # ce tag/commit connu était bon
# Git se place automatiquement au commit du MILIEU
# → testez, puis répondez :
git bisect good                # si le bogue N'EST PAS présent ici
git bisect bad                 # si le bogue EST présent ici
# ... Git répète la division par 2 jusqu'à trouver LE commit fautif
git bisect reset               # termine et revient à votre branche de départ
```

Sur 1000 commits, `git bisect` trouve le coupable en environ **10 tests** (log₂ 1000 ≈ 10), au lieu
de tester chaque commit un à un.

> 💡 **Bonus automatisation** : `git bisect run ./test-du-bogue.sh` exécute un script qui retourne
> un code de sortie 0 (bon) ou différent de 0 (mauvais) — Git bisecte alors **tout seul**, sans
> intervention manuelle à chaque étape.

Ce principe — diviser un espace de recherche en deux pour converger en `log(n)` étapes — est
formalisé par Andreas Zeller sous le nom de *delta debugging* dans *Why Programs Fail* (déjà vu en
semaine 2) : `git bisect` en est une application directe, appliquée à l'historique des commits
plutôt qu'aux données d'entrée.

{{% notice tip "🧪 À vous de jouer — sur votre projet" %}}
Choisissez un fichier avec plusieurs commits d'historique. Simulez un `git bisect` : identifiez un
commit « bon » ancien et le commit actuel comme point de départ, puis parcourez manuellement (ou
avec `git bisect start`) pour retrouver un commit précis où un comportement a changé.
{{% /notice %}}

---

## 📰 Actualité de l'industrie

- **Débogage assisté par IA** : les IDE modernes (GitHub Copilot, IntelliJ AI Assistant) peuvent
  maintenant expliquer une stack trace ou suggérer une hypothèse de cause directement dans la
  session de débogage — un gain de temps, mais qui **remplace rarement** la nécessité de vérifier
  l'hypothèse vous-même (elle peut se tromper avec assurance).
- **Observabilité en production** : des outils comme **Sentry**, **Datadog** ou **New Relic**
  appliquent la même logique « reproduire → isoler → hypothèse » à grande échelle, en capturant
  automatiquement stack traces, variables et contexte au moment d'un crash en production — sans
  breakpoint possible sur un serveur en direct.
- Plusieurs études sur les pratiques de développement (dont le rapport *Developer Coefficient*,
  Stripe, 2018) estiment qu'une part importante du temps des développeur·euse·s est consacrée à la
  correction de bogues et de dette technique plutôt qu'à écrire du nouveau code — un rappel
  concret de l'importance de maîtriser ces outils tôt dans votre formation.

---

## 🧭 Petit résumé — quel outil pour quelle situation ?

| Situation | Outil à privilégier |
|---|---|
| Suivre un flux d'exécution précis, étape par étape | Débogueur, *step over/into* |
| Un cas précis au milieu d'une grande collection/boucle | Breakpoint conditionnel |
| Comportement à observer sur plusieurs exécutions, ou en production | Journalisation (`logger.debug`/`info`) |
| Cause « quelque part » dans beaucoup de code ou de commits | Bissection (manuelle ou `git bisect`) |

## 📚 Références

- Andreas Zeller (2009), *Why Programs Fail: A Guide to Systematic Debugging*, 2ᵉ édition, Morgan
  Kaufmann — le concept de *delta debugging* qui fonde la bissection.
- Documentation officielle JetBrains —
  [Debug code (IntelliJ IDEA)](https://www.jetbrains.com/help/idea/debugging-code.html) : la
  démarche générale de débogage (breakpoints → run in debug → examiner l'état → stepping) ainsi
  que les Watches et *Evaluate Expression* présentés plus haut en sont directement tirés.
- Documentation officielle Git — [`git bisect`](https://git-scm.com/docs/git-bisect).
- Documentation SLF4J — [niveaux de journalisation](https://www.slf4j.org/manual.html).
