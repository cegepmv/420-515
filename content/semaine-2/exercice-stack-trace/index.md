+++
title = "Exercice : lire une stack trace"
weight = 4
+++

## 🎯 Objectif

Mettre en pratique la stratégie de lecture d'une pile d'appels (stack trace) vue dans
[Techniques de débogage (1/2)]({{< relref "/semaine-2/debogage" >}}) : trouver la cause racine,
localiser le point d'entrée pertinent dans votre propre code, et formuler une hypothèse de cause.

## 📝 Déroulement

Pour chacune des piles d'appels ci-dessous, répondez à trois questions avant de révéler la
réponse :

1. Où l'exception a-t-elle été **levée en premier** (fichier + ligne) ?
2. Y a-t-il un `Caused by:` ? Si oui, quelle est la **cause racine** ?
3. Quelle **hypothèse** feriez-vous sur la cause du problème ?

---

**Stack trace 1**

```
java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 5
    at ca.cegepmv.legacy.service.FactureService.calculerTotal(FactureService.java:31)
    at ca.cegepmv.legacy.service.FactureService.genererFacture(FactureService.java:14)
    at ca.cegepmv.legacy.controller.FactureController.creer(FactureController.java:22)
```

<details>
<summary>✅ Voir la réponse</summary>

1. `FactureService.java:31`, dans `calculerTotal`.
2. Aucun `Caused by:` — c'est directement le point d'origine.
3. Hypothèse plausible : une boucle utilise `<=` au lieu de `<` en itérant sur un tableau de 5
   éléments (indices valides 0 à 4), ou un index est calculé à partir d'une valeur externe sans
   validation de ses bornes.
</details>

---

**Stack trace 2**

```
org.springframework.dao.DataIntegrityViolationException: could not execute statement
    at ca.cegepmv.legacy.repository.ClientRepository.save(ClientRepository.java:48)
    at ca.cegepmv.legacy.service.ClientService.creerClient(ClientService.java:19)
    at ca.cegepmv.legacy.controller.ClientController.creer(ClientController.java:18)
Caused by: java.sql.SQLIntegrityConstraintViolationException: Duplicate entry 'jdupont@mail.com' for key 'clients.uk_courriel'
    at ca.cegepmv.legacy.repository.ClientRepository.insert(ClientRepository.java:55)
    ... 3 more
```

<details>
<summary>✅ Voir la réponse</summary>

1. `ClientRepository.java:48` est la première ligne de votre propre code, mais la cause racine se
   trouve plus bas.
2. Oui — `Caused by: SQLIntegrityConstraintViolationException: Duplicate entry ...` : la vraie
   cause est une **violation de contrainte d'unicité** sur la colonne `courriel`.
3. Hypothèse plausible : le service ne valide pas, avant l'insertion, qu'un client avec ce
   courriel existe déjà — il faudrait ajouter une vérification (ou gérer l'exception) avant
   d'appeler `save`.
</details>

---

**Stack trace 3**

```
java.lang.NullPointerException: Cannot invoke "ca.cegepmv.legacy.model.Adresse.getVille()" because the return value of "ca.cegepmv.legacy.model.Client.getAdresse()" is null
    at ca.cegepmv.legacy.service.LivraisonService.calculerFrais(LivraisonService.java:63)
    at ca.cegepmv.legacy.service.CommandeService.valider(CommandeService.java:40)
    at ca.cegepmv.legacy.controller.CommandeController.creer(CommandeController.java:25)
```

<details>
<summary>✅ Voir la réponse</summary>

1. `LivraisonService.java:63`, dans `calculerFrais`.
2. Aucun `Caused by:`, mais le message est très précis (Java moderne) : `getAdresse()` retourne
   `null`.
3. Hypothèse plausible : certains clients existants en base n'ont jamais eu d'adresse enregistrée
   (donnée historique incomplète), et `calculerFrais` ne vérifie pas que `getAdresse()` est
   non-null avant de l'utiliser.
</details>

---

## ✅ Auto-vérification

- [ ] J'ai su distinguer, dans chaque exemple, la première ligne affichée de la **vraie cause
      racine** quand un `Caused by:` était présent.
- [ ] Mes hypothèses étaient formulées de façon **vérifiable** (pas juste « il y a un bogue
      quelque part »).
- [ ] Je comprends pourquoi corriger uniquement le symptôme (ex. un `if (x != null)` ajouté au
      hasard) ne réglerait pas nécessairement la cause réelle dans les scénarios 2 et 3.

> 💡 En équipe, comparez vos hypothèses avec celles de vos coéquipiers — il est normal d'avoir des
> pistes légèrement différentes tant qu'elles sont toutes vérifiables.
