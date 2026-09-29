+++
title = "Réusinage (refactoring) et code smells"
weight = 1
draft = false
+++

## 1️⃣ Qu'est-ce que le réusinage — et ce que ce n'est PAS

Martin Fowler définit le **réusinage** (*refactoring*) ainsi :

> « A change made to the internal structure of software to make it easier to understand and
> cheaper to modify without changing its observable behavior. »
> — Martin Fowler, *Refactoring: Improving the Design of Existing Code* (2e éd., 2019)

Trois éléments de cette définition sont essentiels, et souvent mal compris en pratique :

1. **Structure interne seulement** — on réorganise le code (noms, découpage en méthodes/classes,
   élimination de duplication), on ne touche pas à *ce qu'il fait*.
2. **Comportement observable inchangé** — pour n'importe quelle entrée valide, la sortie doit
   rester **strictement identique** avant et après le réusinage. Si le comportement change, ce
   n'est plus un réusinage : c'est soit une correction de bogue, soit une nouvelle fonctionnalité.
3. **Plus facile à comprendre et moins coûteux à modifier** — le réusinage n'est jamais une fin en
   soi ; il se justifie uniquement s'il rend le prochain changement (fonctionnalité ou correctif)
   plus simple, plus rapide ou moins risqué.

### Réusinage vs correction de bogue vs nouvelle fonctionnalité

| Type de changement | Le comportement observable change-t-il ? | Objectif |
|---|---|---|
| **Réusinage** | Non — jamais | Améliorer la structure interne |
| **Correction de bogue** | Oui — un comportement incorrect devient correct | Faire correspondre le comportement à la spécification |
| **Nouvelle fonctionnalité** | Oui — un nouveau comportement apparaît | Répondre à un nouveau besoin |

---

## 2️⃣ Pourquoi la suite de tests est la condition préalable

Réusiner sans filet de tests, c'est réorganiser le code **en espérant** ne rien casser — on ne le
sait qu'en observant si quelqu'un se plaint plus tard, en production. Avec une suite de tests
automatisés qui couvre déjà le comportement du code (semaine 5), chaque petite étape de réusinage
peut être **validée immédiatement** :

```
1. Les tests passent (état de départ, vert).
2. Appliquer UNE petite transformation (ex. extraire une méthode).
3. Relancer la suite de tests.
4. Vert → committer cette étape, passer à la suivante.
   Rouge → annuler cette étape (ou corriger immédiatement), ne JAMAIS accumuler
           plusieurs transformations non vérifiées.
```

> 💡 C'est le même réflexe que le cycle **Red-Green-Refactor** du développement piloté par les
> tests (Kent Beck) — sauf qu'ici, on entre directement dans la phase *Refactor*, avec des tests
> déjà au vert, écrits à la semaine précédente. Le pipeline d'intégration continue (semaine 5)
> rejoue cette vérification à chaque `push`, ce qui protège aussi vos coéquipiers d'une régression
> que vous auriez introduite sans vous en rendre compte.

Cela justifie un principe simple, mais souvent ignoré en pratique : **ne réusinez jamais du code
qui n'est couvert par aucun test.** Si le code que vous voulez réusiner n'a pas de test, la
première étape consiste à **en écrire un** qui capture le comportement actuel (même si ce
comportement contient un bogue connu — un test de caractérisation documente ce que le code fait
réellement, pas ce qu'il devrait faire), *avant* de toucher à la structure.

<details>
<summary>🤔 Testez-vous</summary>

Un coéquipier propose de réusiner une classe qui n'a aucun test, en argumentant qu'« on ira plus
vite en ne perdant pas de temps à écrire des tests d'abord ». Que répondez-vous ?

**Réponse** : sans tests, il n'y a aucun moyen automatisé de confirmer que le comportement
observable n'a pas changé pendant le réusinage — chaque transformation devient un pari plutôt
qu'une vérification. Le temps « gagné » en sautant les tests est perdu (souvent avec intérêt) au
premier bogue de régression découvert en production, sans historique clair pour le diagnostiquer.
Écrire d'abord un test de caractérisation qui capture le comportement actuel est un prérequis, pas
une perte de temps.
</details>

---

## 3️⃣ Catalogue de *code smells* courants

Un **code smell** (terme popularisé par Kent Beck et Martin Fowler) est un indice **de surface** —
pas une preuve formelle — qu'un problème de structure plus profond existe probablement quelque
part dans le code. Un smell n'est pas un bogue : le code smellé peut très bien fonctionner
correctement ; il est simplement plus difficile à comprendre, tester ou faire évoluer qu'il ne
devrait l'être.

> ⚠️ **Attention aux nombres magiques dans les exemples ci-dessous.** Les blocs marqués ❌ *Avant*
> contiennent volontairement des constantes non nommées (`250.0`, `0.02`, `1.5`...) : c'est
> justement une partie du problème à corriger, pas seulement le smell mis en vedette dans chaque
> section. Un nombre magique **sans commentaire ni nom** est en soi un défaut — même dans un
> exemple « corrigé » qui se concentre sur un autre smell. Les blocs ✅ *Après* ci-dessous nomment
> donc systématiquement ces constantes (`SURGERY_RATE`, `LATE_PAYMENT_PENALTY_RATE`, etc.), même
> quand ce n'est pas le smell principal illustré — voir aussi *Replace Magic Number* en section 4️⃣
> pour ce réusinage en détail.

### Méthode trop longue (*Long Method*)

Une méthode qui fait « plusieurs choses » (validation, calcul, formatage, journalisation) est
difficile à nommer précisément, à tester unitairement, et à réutiliser en partie.

```java
// ❌ Avant : une seule méthode qui mélange validation, calcul et formatage
public String buildInvoiceSummary(Owner owner, List<Visit> visits) {
    if (owner == null || visits == null || visits.isEmpty()) {
        throw new IllegalArgumentException("Owner and visits are required");
    }
    double total = 0;
    for (Visit visit : visits) {
        double base = visit.getType().equals("SURGERY") ? 250.0 : 60.0;
        total += base * (visit.isEmergency() ? 1.5 : 1.0);
    }
    StringBuilder sb = new StringBuilder();
    sb.append("Invoice for ").append(owner.getFirstName()).append(" ").append(owner.getLastName());
    sb.append(" — Total: $").append(String.format("%.2f", total));
    return sb.toString();
}
```

```java
private static final double SURGERY_RATE = 250.0;
private static final double STANDARD_RATE = 60.0;
private static final double EMERGENCY_MULTIPLIER = 1.5;

// ✅ Après : la validation et le calcul (qui contient une boucle) sont extraits ;
// le formatage, un simple one-liner utilisé une seule fois, reste inline
public String buildInvoiceSummary(Owner owner, List<Visit> visits) {
    validateInvoiceInput(owner, visits);
    double total = calculateTotal(visits);
    return "Invoice for %s %s — Total: $%.2f".formatted(
            owner.getFirstName(), owner.getLastName(), total);
}

private void validateInvoiceInput(Owner owner, List<Visit> visits) {
    if (owner == null || visits == null || visits.isEmpty()) {
        throw new IllegalArgumentException("Owner and visits are required");
    }
}

private double calculateTotal(List<Visit> visits) {
    double total = 0;
    for (Visit visit : visits) {
        double base = visit.getType().equals("SURGERY") ? SURGERY_RATE : STANDARD_RATE;
        total += base * (visit.isEmergency() ? EMERGENCY_MULTIPLIER : 1.0);
    }
    return total;
}

```

> 💡 **N'extrayez pas systématiquement** : une méthode privée n'a de valeur que si elle apporte
> quelque chose — un nom qui clarifie l'intention d'un bloc autrement peu lisible, une logique
> testable isolément, ou l'élimination d'une duplication. Ici, `validateInvoiceInput` et
> `calculateTotal` méritent d'être extraites (la première nomme une intention, la seconde contient
> une boucle avec sa propre logique) ; par contre, transformer le simple `"Invoice for ...".
> formatted(...)` en une méthode `formatSummary` séparée n'ajoute rien — c'est déjà lisible tel
> quel, et créer une méthode d'une seule ligne, appelée à un seul endroit, ne fait qu'ajouter un
> niveau d'indirection à suivre pour le prochain lecteur. Extraire pour extraire n'est pas
> l'objectif ; réduire l'effort de compréhension l'est.

**🔧 Comment corriger** : appliquer **Extract Method** — découper la méthode en sous-méthodes
privées nommées, une par responsabilité (valider, calculer), **en jugeant au cas par cas
si chaque extraction en vaut la peine**. Règle pratique : si vous devez ajouter un commentaire du
genre `// maintenant on calcule le total` avant un bloc de plusieurs lignes (surtout avec une
boucle ou une condition), ce bloc est probablement une méthode qui s'ignore elle-même — extrayez-le
et laissez le nom de la méthode remplacer le commentaire. Mais si le bloc tient déjà en une seule
ligne claire, l'extraction ajoute plus de bruit (indirection, saut vers une autre méthode) que de
clarté.

### Duplication de code (*Duplicated Code*)

La même logique copiée-collée à plusieurs endroits — un correctif ou une évolution doit alors être
appliqué **partout**, et il suffit d'un seul oubli pour introduire une incohérence.

```java
// ❌ Avant : le même calcul de tarif est recopié dans 3 méthodes différentes
public double genererFacture(Visit visit) {
    double base = visit.getType().equals("SURGERY") ? 250.0 : 60.0; // dupliqué
    return base * (visit.isEmergency() ? 1.5 : 1.0);
}

public double estimerCoutAnnuel(List<Visit> visits) {
    double total = 0;
    for (Visit v : visits) {
        total += v.getType().equals("SURGERY") ? 250.0 : 60.0; // dupliqué
    }
    return total * 12;
}
```

```java
// ✅ Après : la logique de tarif existe à un seul endroit, réutilisée partout,
// et les nombres magiques sont eux aussi nommés (voir Replace Magic Number, section 4️⃣)
private static final double SURGERY_RATE = 250.0;
private static final double STANDARD_RATE = 60.0;
private static final double EMERGENCY_MULTIPLIER = 1.5;

private double calculerTarifBase(Visit visit) {
    return visit.getType().equals("SURGERY") ? SURGERY_RATE : STANDARD_RATE;
}

public double genererFacture(Visit visit) {
    return calculerTarifBase(visit) * (visit.isEmergency() ? EMERGENCY_MULTIPLIER : 1.0);
}

public double estimerCoutAnnuel(List<Visit> visits) {
    double total = 0;
    for (Visit v : visits) {
        total += calculerTarifBase(v);
    }
    return total * 12;
}
```

**🔧 Comment corriger** : appliquer **Extract Method** (ou **Extract Class** si la logique
dupliquée est assez riche pour devenir un concept à part entière — voir *Repeated Switch/If
Chains* ci-dessous) pour centraliser la logique à un seul endroit, puis faire appeler cette
méthode/classe partout où le duplicata existait. Utilisez la fonctionnalité *Find Usages*/*Find
Similar Code* de l'IDE pour vous assurer d'avoir bien repéré **toutes** les occurrences avant de
supprimer les copies.

### Classe trop grosse (*Large Class* / *God Class*)

Une classe qui accumule trop de responsabilités (souvent reconnaissable à un grand nombre de
champs et de méthodes sans lien évident entre eux) devient un point de contention : presque tout le
monde doit la modifier, ce qui multiplie les conflits de fusion et les effets de bord imprévus.

```java
// ❌ Avant : Owner accumule l'identité, la facturation, les notifications, la fidélité
// ET la génération de rapports — bien plus que 3-4 champs qui voyagent ensemble
// (contrairement à Data Clumps ci-dessus, ici ce sont des COMPORTEMENTS non liés qui
// s'accumulent, pas seulement des données)
public class Owner {
    private String firstName, lastName, email;
    private double balance;
    private int loyaltyPoints;
    private List<Pet> pets;

    public double calculateLateFee() {
        return balance < 0 ? Math.abs(balance) * 0.02 : 0;
    }

    public void sendReminderEmail(String message) {
        // ... construit et envoie un courriel via un client SMTP ...
    }

    public int calculateLoyaltyTier() {
        return pets.size() > 1 && balance >= 0 ? 2 : 1;
    }

    public String generateAnnualStatement() { /* ... 40 lignes de mise en page ... */ return ""; }
}
```

```java
// ✅ Après : chaque responsabilité migre vers une classe dédiée à un seul rôle ;
// Owner ne conserve plus que ses propres données (identité, solde, animaux)
public class Owner {
    private String firstName, lastName, email;
    private double balance;
    private int loyaltyPoints;
    private List<Pet> pets;
    // getters uniquement — plus aucune logique de facturation/notification/fidélité ici
}

public class BillingService {
    private static final double LATE_PAYMENT_PENALTY_RATE = 0.02;

    public double calculateLateFee(Owner owner) {
        return owner.getBalance() < 0
                ? Math.abs(owner.getBalance()) * LATE_PAYMENT_PENALTY_RATE
                : 0;
    }
}

public class NotificationService {
    public void sendReminderEmail(Owner owner, String message) {
        // ... construit et envoie un courriel via un client SMTP ...
    }
}

public class LoyaltyService {
    public int calculateTier(Owner owner) {
        return owner.getPets().size() > 1 && owner.getBalance() >= 0 ? 2 : 1;
    }
}

public class StatementService {
    public String generateAnnualStatement(Owner owner) { /* ... */ return ""; }
}
```

**🔧 Comment corriger** : appliquer **Extract Class**, mais ici pour séparer des
**responsabilités** (facturation, notifications, fidélité, rapports), pas seulement des champs
qui voyagent ensemble — c'est la différence clé avec *Data Clumps*. Un bon indice de *God Class* :
la classe a besoin d'être importée par des modules qui n'ont rien en commun entre eux (le module
de facturation, celui des notifications, celui des statistiques...) simplement parce que toute la
logique de l'application y est empilée. Après le réusinage, `Owner` redevient un simple porteur de
données ; chaque service ne dépend que des données `Owner` dont il a réellement besoin.

### Liste de paramètres trop longue (*Long Parameter List*)

```java
// ❌ Avant : 6 paramètres, ordre facile à inverser par erreur
public Visit scheduleVisit(int petId, LocalDate date, String type,
                           boolean emergency, String vetName, String notes) { ... }
```

```java
// ✅ Après : les paramètres qui voyagent toujours ensemble deviennent un seul objet
public record VisitRequest(int petId, LocalDate date, String type,
                            boolean emergency, String vetName, String notes) {}

public Visit scheduleVisit(VisitRequest request) { ... }
```

**🔧 Comment corriger** : appliquer **Introduce Parameter Object** — regrouper les paramètres en
un `record` (ou une petite classe immuable) nommé selon le concept qu'il représente. Bénéfice
secondaire : si un 7e paramètre doit être ajouté plus tard, un seul type change, au lieu de devoir
modifier la signature (et tous les appels) de la méthode.

### Obsession des primitifs (*Primitive Obsession*)

Utiliser des `String`/`int` bruts pour représenter un concept du domaine (un email, un numéro de
téléphone, un montant monétaire) plutôt qu'un petit type dédié — rien n'empêche alors de passer un
email là où un nom est attendu, puisque ce sont tous les deux des `String` pour le compilateur.

```java
// ❌ Avant : rien n'empêche d'inverser email et téléphone par erreur — les deux sont des String
public void notifyOwner(String email, String phone, String message) { ... }
notifyOwner(phoneNumber, ownerEmail, "Rappel de vaccin"); // compile, mais c'est inversé !
```

```java
// ✅ Après : des types dédiés rendent l'inversion impossible à la compilation
public record Email(String value) { }
public record PhoneNumber(String value) { }

public void notifyOwner(Email email, PhoneNumber phone, String message) { ... }
```

**🔧 Comment corriger** : appliquer **Replace Primitive with Object** (aussi appelé *Introduce
Value Object*) — envelopper la primitive dans un petit type dédié (souvent un `record`), qui peut
au passage valider son propre format (ex. un `Email` refuse une chaîne sans `@` dans son
constructeur) plutôt que de laisser cette responsabilité éparpillée dans tout le code appelant.

> 🤔 **Pourquoi pas simplement `notifyOwner(email = "samuel@...", phone = "...")` ?** Dans des
> langages comme Python ou Kotlin, les **arguments nommés** à l'appel permettraient effectivement
> d'éviter l'inversion (`notifyOwner(phone = ownerPhone, email = ownerEmail)` resterait correct
> même dans le mauvais ordre). **Java ne supporte pas les arguments nommés** — à l'appel, seul
> l'ordre positionnel compte, et rien dans le nom du paramètre à la déclaration n'est vérifié ni
> visible pour le compilateur. C'est justement pour cette raison qu'en Java, on compense ce
> manque de garde-fou du langage en donnant un **type distinct** à chaque concept (`Email`,
> `PhoneNumber`) : le compilateur peut alors détecter l'inversion, même sans arguments nommés.

### Envie de fonctionnalité (*Feature Envy*)

Une méthode qui s'intéresse **davantage** aux données d'un autre objet qu'aux siennes propres — un
signal que cette logique devrait probablement vivre dans l'autre classe.

```java
// ❌ Avant : PetService manipule presque exclusivement les champs internes de Owner
public class PetService {
    public String describeOwner(Owner owner) {
        return owner.getFirstName() + " " + owner.getLastName()
             + " (" + owner.getAddress() + ", " + owner.getCity() + ")";
    }
}
```

```java
// ✅ Après : la logique déménage vers la classe dont elle utilise réellement les données
public class Owner {
    public String describe() {
        return firstName + " " + lastName + " (" + address + ", " + city + ")";
    }
}

public class PetService {
    public String describeOwner(Owner owner) {
        return owner.describe();
    }
}
```

**🔧 Comment corriger** : appliquer **Move Method** — déplacer la méthode (ou la partie du corps
concernée) vers la classe dont elle utilise le plus les données. Astuce pour la repérer : si une
méthode appelle plus de *getters* sur un paramètre que sur `this`, c'est un signal fort qu'elle est
mal placée.

### Groupes de données (*Data Clumps*)

Les mêmes 3-4 champs (ex. `street`, `city`, `postalCode`) qui voyagent systématiquement ensemble en
paramètres, sans jamais être regroupés en un seul objet — un signal qu'ils forment en réalité un
concept manquant (ici, une `Address`).

```java
// ❌ Avant : street/city/postalCode voyagent toujours ensemble, séparément
public class Owner {
    private String street, city, postalCode;
}
public void shipInvoice(String street, String city, String postalCode) { ... }
```

```java
// ✅ Après : le concept manquant (Address) devient un type explicite, réutilisé partout
public record Address(String street, String city, String postalCode) { }

public class Owner {
    private Address address;
}
public void shipInvoice(Address address) { ... }
```

**🔧 Comment corriger** : appliquer **Extract Class** (pour regrouper les champs dans l'entité) et
**Introduce Parameter Object** (pour regrouper les paramètres équivalents dans les signatures de
méthode) — les deux réusinages s'attaquent au même symptôme, à deux endroits différents du code
(champs vs paramètres).

### Chaînes de conditions répétées (*Repeated Switch/If Chains*)

Le même `switch`/`if-else` sur un même type de donnée, dupliqué à plusieurs endroits du code —
ajouter un nouveau cas oblige à retrouver et modifier chaque occurrence.

```java
// ❌ Avant : ce switch sur visit.getType() est recopié — presque à l'identique —
// dans 3 méthodes différentes, une par comportement qui varie selon le type
public double tarifPour(Visit visit) {
    return switch (visit.getType()) {
        case "SURGERY" -> 250.0;
        case "CHECKUP" -> 60.0;
        case "VACCINATION" -> 35.0;
        default -> throw new IllegalArgumentException("Type inconnu");
    };
}

public int dureeMinutesPour(Visit visit) {
    return switch (visit.getType()) {
        case "SURGERY" -> 90;
        case "CHECKUP" -> 20;
        case "VACCINATION" -> 10;
        default -> throw new IllegalArgumentException("Type inconnu");
    };
}

public String descriptionPour(Visit visit) {
    return switch (visit.getType()) {
        case "SURGERY" -> "Intervention chirurgicale";
        case "CHECKUP" -> "Examen de routine";
        case "VACCINATION" -> "Rappel de vaccin";
        default -> throw new IllegalArgumentException("Type inconnu");
    };
}
```

Ajouter un 4e type de visite (ex. `"DENTAL"`) oblige à retrouver et modifier **ces 3 méthodes** —
et rien n'empêche d'en oublier une : le compilateur ne signale aucune erreur si `dureeMinutesPour`
n'est pas mis à jour, il continuera simplement à lancer `IllegalArgumentException` pour ce cas
oublié, en production.

```java
// ✅ Après (solution recommandée) : un enum à champs regroupe les 3 comportements
// en un seul endroit, avec beaucoup moins de code que des classes séparées
public enum TypeVisite {
    CHIRURGIE(250.0, 90, "Intervention chirurgicale"),
    CONSULTATION(60.0, 20, "Examen de routine"),
    VACCINATION(35.0, 10, "Rappel de vaccin");

    private final double tarif;
    private final int dureeMinutes;
    private final String description;

    TypeVisite(double tarif, int dureeMinutes, String description) {
        this.tarif = tarif;
        this.dureeMinutes = dureeMinutes;
        this.description = description;
    }

    public double tarif() { return tarif; }
    public int dureeMinutes() { return dureeMinutes; }
    public String description() { return description; }
}
// visit.getTypeVisite().tarif() / .dureeMinutes() / .description()
```

Ajouter un 4e type de visite ne demande plus qu'**une seule ligne** (une nouvelle constante
d'enum) — impossible d'oublier un des 3 comportements, puisque le constructeur de l'enum exige les
3 valeurs pour chaque constante.

**🔧 Comment corriger** : quand la variation entre les types se limite à des **données**
constantes (un tarif, une durée, un libellé — comme ici), un **enum à champs** règle le problème
avec beaucoup moins de code qu'une hiérarchie de classes, tout en gardant le même bénéfice
(compilateur qui force à définir les 3 comportements pour chaque type). Réservez **Replace
Conditional with Polymorphism** (interface + une classe par type) au cas où chaque type exige un
**algorithme réellement différent** — pas seulement une valeur différente. Par exemple, si le
calcul du tarif d'une chirurgie devait tenir compte d'une durée variable et d'un supplément de
matériel, alors qu'une vaccination reste un montant fixe, ces deux logiques ne peuvent plus
tenir dans un simple champ d'enum ; chaque type mérite alors sa propre implémentation :

```java
public interface CalculateurTarif {
    double calculer(Visite visite);
}

public class CalculateurTarifChirurgie implements CalculateurTarif {
    public double calculer(Visite visite) {
        return TARIF_BASE_CHIRURGIE
                + visite.getDureeReelleMinutes() * SUPPLEMENT_PAR_MINUTE
                + visite.getFraisMateriel();
    }
}

public class CalculateurTarifVaccination implements CalculateurTarif {
    public double calculer(Visite visite) {
        return TARIF_FIXE_VACCINATION;
    }
}
```

> 💡 Dans ce dernier exemple, chaque méthode `calculer` contient une **vraie logique**, pas
> seulement une valeur de retour — c'est ce qui justifie une classe séparée par type. Si toutes
> les implémentations se réduisaient à `return UNE_CONSTANTE;`, ce serait le signal qu'un enum à
> champs suffit, et que l'interface ajoute de la complexité sans bénéfice réel.

Sur un `switch` qui n'apparaît qu'à un **seul** endroit du code, un simple **Extract Method** qui
centralise ce `switch` dans une méthode partagée reste la solution la plus simple — inutile
d'introduire un enum ou une interface pour un `switch` unique qui ne change presque jamais.

<details>
<summary>🤔 Testez-vous</summary>

Une méthode `printOwnerDetails(Owner owner)` accède à `owner.getPet().getVisits().get(0).getVet()`
et n'utilise presque aucun champ de son propre objet contenant. Quel *code smell* cela illustre-t-il,
et quelle piste de réusinage suggère-t-il ?

**Réponse** : c'est un exemple de *Feature Envy* — la méthode s'intéresse presque exclusivement aux
données d'objets externes (`Pet`, `Visit`, `Vet`) plutôt qu'aux siennes. Cela suggère de déplacer
cette logique plus près des données qu'elle manipule (ex. une méthode sur `Pet` ou `Visit`), ou
d'extraire un objet intermédiaire qui expose directement ce qui est nécessaire, plutôt que de
naviguer une longue chaîne d'appels depuis l'extérieur.
</details>

---

## 4️⃣ Catalogue de réusinages — avant / après

Fowler documente des dizaines de réusinages nommés et catalogués, chacun avec une procédure précise
et réversible. La section précédente a déjà montré, pour chaque smell, la technique qui le corrige
(**Extract Method** pour *Long Method*/*Duplicated Code*, **Extract Class** pour *Large
Class*/*Data Clumps*, **Introduce Parameter Object** pour *Long Parameter List*/*Data Clumps*,
**Move Method** pour *Feature Envy*, **Replace Primitive with Object** pour *Primitive Obsession*,
**Replace Conditional with Polymorphism** pour *Repeated Switch/If Chains*). Voici quatre autres
réusinages tout aussi fréquents, qui ne correspondent pas à un seul smell précis mais s'appliquent
au quotidien, dans presque toutes les méthodes.

### Extraire une variable (*Extract Variable*)

Nommer une sous-expression complexe pour documenter son intention, plutôt que de la laisser inline.

```java
// ❌ Avant
if (visit.isEmergency() && visit.getType().equals("SURGERY") && owner.getBalance() < 0) { ... }

// ✅ Après
boolean isUnpaidEmergencySurgery =
        visit.isEmergency() && visit.getType().equals("SURGERY") && owner.getBalance() < 0;
if (isUnpaidEmergencySurgery) { ... }
```

### Renommer (*Rename Variable/Method/Field*)

Le réusinage le plus fréquent et souvent le plus sous-estimé : un bon nom **remplace un
commentaire**. Toujours effectué via la fonction *rename* de l'IDE (jamais un chercher-remplacer
manuel — voir section 6).

```java
// ❌ Avant
int d; // nombre de jours depuis la dernière visite

// ✅ Après
int daysSinceLastVisit;
```

> 💡 **Convention de nommage pour les méthodes** : une méthode fait quelque chose (elle calcule,
> valide, génère, envoie...) — son nom devrait donc **commencer par un verbe** (`calculerTotal`,
> `validerEntree`, `genererRapport`), jamais par un simple nom commun (`total`, `tarifBase`,
> `rapport`). Un nom sans verbe se confond facilement avec un champ ou une propriété, et cache le
> fait qu'un calcul ou un effet a lieu. Les champs et variables, à l'inverse, sont généralement de
> simples noms (`daysSinceLastVisit`, `total`), sans verbe.

### Remplacer un nombre magique par une constante nommée (*Replace Magic Number with Symbolic Constant*)

```java
// ❌ Avant : que représente 250.0 ? Et si le tarif change, combien d'endroits corriger ?
double base = visit.getType().equals("SURGERY") ? 250.0 : 60.0;

// ✅ Après
private static final double SURGERY_RATE = 250.0;
private static final double STANDARD_RATE = 60.0;

double base = visit.getType().equals("SURGERY") ? SURGERY_RATE : STANDARD_RATE;
```

### Introduire une clause de garde (*Guard Clauses*)

Remplacer un imbriquement profond de `if`/`else` par des sorties anticipées (`return`/`throw`)
réduit la profondeur d'indentation et rend le cas normal (« happy path ») immédiatement visible.

```java
// ❌ Avant : la logique utile est noyée dans l'imbrication
public double calculateDiscount(Owner owner) {
    double discount = 0;
    if (owner != null) {
        if (owner.getBalance() >= 0) {
            if (owner.getPets().size() > 1) {
                discount = 0.1;
            }
        }
    }
    return discount;
}

// ✅ Après : les cas d'exclusion sortent tôt, le cas normal reste au premier niveau,
// et le nombre magique 0.1 devient une constante nommée
private static final double MULTI_PET_DISCOUNT_RATE = 0.1;

public double calculateDiscount(Owner owner) {
    if (owner == null) return 0;
    if (owner.getBalance() < 0) return 0;
    if (owner.getPets().size() <= 1) return 0;
    return MULTI_PET_DISCOUNT_RATE;
}
```

<details>
<summary>🤔 Testez-vous</summary>

Vous repérez le même bloc `if (type.equals("SURGERY")) { rate = 250.0; } else { rate = 60.0; }`
copié dans quatre méthodes différentes d'une même classe. Quel réusinage appliquer en premier —
*Extract Class* ou *Extract Method* ?

**Réponse** : *Extract Method* d'abord — regrouper cette logique dupliquée dans une seule méthode
privée (`private double calculerTarif(String type)`) appelée aux quatre endroits élimine déjà la
duplication avec un changement minimal et réversible. *Extract Class* serait une transformation
plus large, à envisager seulement si cette logique de tarification grossit encore et mérite de
devenir un concept autonome du domaine (ex. une classe `PricingPolicy`) — inutile de sauter
directement à la solution la plus lourde quand une plus légère règle déjà le problème observé.
</details>

---

## 5️⃣ Quand réusiner — et quand s'en abstenir

### La règle du boy-scout (*Boy Scout Rule*)

Attribuée à Robert C. Martin : « laissez le camping plus propre que vous ne l'avez trouvé ». En
pratique : chaque fois que vous touchez à un fichier pour une autre raison (corriger un bogue,
ajouter une fonctionnalité), profitez-en pour améliorer légèrement ce que vous voyez au passage
(renommer une variable mal nommée, extraire une méthode trop longue) — sans transformer ce
changement en un réusinage massif et hors sujet par rapport à la tâche en cours.

### Quand reporter un réusinage

- **Le code n'est couvert par aucun test** — écrivez d'abord un test de caractérisation (section 2)
  avant de toucher à la structure.
- **Une échéance très serrée approche** — un réusinage risqué juste avant une livraison est un pari
  dont le potentiel désavantage (régression de dernière minute) dépasse largement le bénéfice.
- **Le code est sur le point d'être supprimé ou remplacé** — investir du temps à améliorer la structure d'un code qui disparaîtra bientôt n'apporte aucun bénéfice futur.

---

## 6️⃣ Réusiner par petites étapes, avec les outils de l'IDE

Un réusinage manuel (chercher-remplacer, copier-coller) introduit facilement des erreurs qu'un
outil automatisé évite par construction : la plupart des IDE modernes (IntelliJ IDEA, Eclipse,
VS Code avec les bonnes extensions) offrent des commandes de réusinage **sûres** — elles analysent
le code (pas seulement le texte) avant d'appliquer la transformation, ce qui évite par exemple de
renommer une variable dans un commentaire ou une chaîne de caractères sans lien avec le symbole
visé.

| Réusinage | Raccourci typique (IntelliJ IDEA) |
|---|---|
| Renommer (*Rename*) | `Shift+F6` |
| Extraire une méthode (*Extract Method*) | `Ctrl+Alt+M` |
| Extraire une variable (*Extract Variable*) | `Ctrl+Alt+V` |
| Extraire une constante (*Extract Constant*) | `Ctrl+Alt+C` |
| Déplacer une classe/méthode (*Move*) | `F6` |

Procédure recommandée pour un réusinage de taille non triviale :

1. Confirmer que la suite de tests est **verte** avant de commencer.
2. Appliquer **une seule** transformation catalogée (ex. un seul *Extract Method*), de préférence
   via la commande de l'IDE plutôt que manuellement.
3. Relancer la suite de tests immédiatement.
4. Si verte : committer cette étape avec un message clair (ex. « Extract calculateTotal() from
   buildInvoiceSummary() ») avant de passer à la transformation suivante.
5. Si rouge : ne pas empiler une correction par-dessus — annuler la transformation (`Ctrl+Z` ou
   `git checkout`) et recommencer plus petit.

> 💡 Une série de petits commits de réusinage, chacun accompagné de tests verts, est bien plus
> facile à relire (et à annuler sélectivement si besoin, via `git revert`) qu'un unique commit
> massif « Refactoring » qui mélange dix transformations différentes.

---

## 📚 Références

- Martin Fowler (2019), *Refactoring: Improving the Design of Existing Code* (2e éd.),
  Addison-Wesley.
- Robert C. Martin (2009), *Clean Code: A Handbook of Agile Software Craftsmanship*, Prentice Hall.
- Kent Beck (2002), *Test-Driven Development: By Example*, Addison-Wesley — cycle Red-Green-Refactor.
- Catalogue en ligne (extraits du livre de Fowler, mis à jour) —
  [refactoring.com/catalog](https://refactoring.com/catalog/).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Réusinage]({{% relref "/semaine-6/exercice-refactoring" %}}) pour
appliquer ce catalogue directement sur votre projet, avec votre suite de tests comme filet de
sécurité.
{{% /notice %}}
