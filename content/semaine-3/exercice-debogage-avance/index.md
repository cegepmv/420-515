+++
title = "Exercice : outils de débogage"
weight = 2
+++

## 🎯 Objectif

Mettre en pratique, sur votre propre projet, les 4 outils vus dans
[Techniques de débogage (2/2)]({{% relref "/semaine-3/debogage-avance" %}}) : débogueur pas à pas,
breakpoint conditionnel, journalisation stratégique, bissection.

> 👥 **En équipe.** Chaque partie peut être faite par une personne différente, puis partagée en
> équipe — l'objectif est que tout le monde ait manipulé chaque outil au moins une fois.

---

## 🧩 Partie 1 — Débogueur pas à pas

1. Choisissez une méthode de votre projet qui contient une boucle ou une condition non triviale.
2. Posez un breakpoint au début de cette méthode, démarrez en mode Debug, et déclenchez-la
   réellement (appel API, test, ou exécution de l'application).
3. Utilisez *Step Over* pour avancer ligne par ligne, et *Step Into* au moins une fois pour entrer
   dans une méthode appelée.
4. Notez : une valeur de variable qui vous a surpris, et le nom exact de la classe concrète
   affichée par le débogueur si une interface est impliquée.

---

## 🎯 Partie 2 — Breakpoint conditionnel

1. Repérez une collection ou une boucle qui traite plusieurs éléments (ex. une liste retournée par
   un endpoint, ou une boucle `for` sur un tableau).
2. Posez un breakpoint conditionnel qui ne s'arrête que sur un élément précis (par identifiant, par
   une valeur nulle, ou par une valeur hors de l'intervalle attendu).
3. Vérifiez que le programme s'exécute normalement jusqu'à ce cas précis, sans s'arrêter avant.

<details>
<summary>❓ Question</summary>

Pourquoi un breakpoint conditionnel est-il particulièrement utile pour un bogue qui **ne se produit
que sur certaines données**, plutôt qu'un breakpoint classique ?

**Réponse** : un breakpoint classique s'arrête à **chaque** itération, ce qui est trop lent quand
seule une itération précise (parmi des centaines) est en cause. Le breakpoint conditionnel filtre
automatiquement pour vous amener directement au bon endroit.
</details>

---

## 📝 Partie 3 — De `println` à la journalisation

1. Trouvez (ou ajoutez temporairement) un `System.out.println` de débogage dans votre projet.
2. Remplacez-le par un appel à un logger (`logger.debug(...)` ou équivalent dans votre langage),
   avec un message qui inclut le contexte utile (identifiants, valeurs pertinentes).
3. Vérifiez que le message apparaît bien dans la console/les logs au niveau `DEBUG`.

> 💡 Si votre langage/framework n'utilise pas SLF4J (ex. Python, JavaScript), utilisez l'équivalent
> local (`logging` en Python, un logger structuré en Node.js) — le principe des niveaux reste le
> même.

---

## 🔍 Partie 4 — Bissection

**Option A — dans le code** : sur un comportement que vous ne comprenez pas bien, commentez la
moitié d'un bloc suspect, relancez le scénario, et notez si le comportement change. Répétez sur la
moitié restante.

**Option B — dans l'historique (`git bisect`)** : sur votre projet, identifiez un ancien commit où
un comportement précis était différent d'aujourd'hui (ex. un test qui passait avant et échoue
maintenant, ou l'inverse). Utilisez `git bisect start`, `git bisect good`/`bad` pour retrouver le
commit exact où le comportement a changé.

```bash
git bisect start
git bisect bad                 # HEAD (ou un commit récent) présente le problème
git bisect good <ancien-hash>  # ce commit ne présentait pas le problème
# testez à chaque arrêt, puis :
git bisect good   # ou
git bisect bad
# ... jusqu'au commit fautif
git bisect reset
```

---

## ✅ Auto-vérification

- [ ] J'ai utilisé *Step Over* et *Step Into* au moins une fois chacun sur du vrai code.
- [ ] Un breakpoint conditionnel m'a permis d'atteindre directement un cas précis dans une
      collection, sans m'arrêter à chaque itération.
- [ ] J'ai remplacé au moins un `println` par un appel de logger avec le bon niveau.
- [ ] J'ai réduit un espace de recherche par bissection (code ou `git bisect`) jusqu'à isoler une
      seule cause.

> 💡 Gardez vos notes de cette pratique — elles pourront être utiles pour un travail pratique à
> venir.
