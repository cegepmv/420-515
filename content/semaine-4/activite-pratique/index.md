+++
title = "Activité pratique"
weight = 5
+++

## 🎯 Activité pratique (non notée)

En équipe, sur votre application legacy :

1. Écrivez **au moins 3 tests JUnit** sur des classes de votre projet qui n'ont actuellement aucun
   test, en utilisant `@BeforeEach` si plusieurs tests partagent une préparation commune.
2. Exécutez votre suite complète avec `mvn test` et consultez les rapports générés dans
   `target/surefire-reports/`.
3. Ciblez ensuite un seul de vos tests avec `-Dtest=NomDeLaClasse#nomDeLaMéthode` pour confirmer que
   vous savez l'exécuter isolément.

> 💡 Cette activité prépare directement la semaine 5, où ces mêmes tests seront distingués entre
> tests unitaires et tests de régression, mesurés en couverture, puis intégrés à un pipeline
> d'intégration continue.

### 🧭 Exemple concret : progresser du plus facile au plus difficile

La classe `Owner` (`model/Owner.java`) du projet n'a **aucun test** actuellement, alors qu'elle
contient plusieurs comportements intéressants — et pas besoin de mock pour la tester (c'est un
objet du domaine, sans Spring, sans base de données). C'est un bon point de départ. Voici une
progression suggérée, du plus simple au plus subtil :

**Niveau 1 — assertions simples, un seul comportement**

1. `shouldReturnEmptyListWhenOwnerHasNoPets` — un owner neuf → `getPets()` retourne une liste vide
   (pas `null`).
2. `shouldAddPetAndLinkBackToOwner` — après `addPet(pet)`, `pet.getOwner() == owner`.
3. `shouldFindPetByNameIgnoringCase` — `getPet("REX")` trouve un pet nommé `"Rex"`.
4. `shouldReturnNullWhenPetNameNotFound` — `getPet("inconnu")` retourne `null`.
5. `shouldReturnNullWhenPetIdNotFound` — `getPet(9999)` retourne `null`.
6. `toStringShouldContainOwnerFields` — `toString()` contient l'adresse, la ville et le téléphone.

**Niveau 2 — comportements composés (plusieurs pets, `assertAll`/`assertThrows`)**

7. `shouldReturnPetsSortedByNameIgnoringCase` — 3 pets ajoutés dans le désordre → `getPets()` les
   retourne triés par nom, insensible à la casse.
8. `getPetsShouldReturnUnmodifiableList` — `assertThrows(UnsupportedOperationException.class, () ->
   getPets().add(...))`.
9. `shouldIgnoreNewPetsWhenFlagIsTrue` — `getPet(name, true)` ignore un pet dont `isNew()` est vrai.
10. `shouldIncludeNewPetsWhenFlagIsFalse` — `getPet(name, false)` retrouve ce même pet neuf.

**Niveau 3 — cas limite / bug réel (le plus subtil)**

11. `shouldFindPetByIdWithinIntegerCacheRange` — `getPet(1)` fonctionne (petit id, dans le cache
    `Integer` de la JVM).
12. `shouldExposeIntegerCacheBugForLargeIds` — avec un id `> 127`, `getPet(id)` échoue à cause
    d'une comparaison `==` sur des `Integer` autoboxés au lieu de `.equals(...)`. Ce test échoue
    **volontairement** avec le code actuel : c'est l'occasion de faire le pont avec le débogage
    (semaine 3) et de discuter « le test est-il faux, ou le code est-il faux ? ».

{{% notice tip "🧪 À vous de jouer" %}}
Reproduisez cette progression sur une classe de **votre** projet qui n'a aucun test : commencez
par 2-3 cas simples (niveau 1, sans mock), ajoutez des cas qui combinent plusieurs éléments ou
vérifient une exception (niveau 2), puis terminez par un cas limite qui pourrait révéler un bogue
caché (niveau 3).
{{% /notice %}}

> 📌 La semaine 4 est évaluée par le **Quiz 2** (3 points) — Maven (dépendances, cycle de vie,
> exécution des tests) et les bases de JUnit (annotations, assertions, cycle de vie d'une classe de
> test).
