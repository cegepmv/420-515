+++
title = "Analyse statique avec SonarQube / SonarCloud"
weight = 1
draft = false
+++

Tout ce qui suit est fondé sur la documentation officielle **SonarQube Cloud** (la version hébergée
et gratuite pour les projets open source de SonarSource) — voir
[docs.sonarsource.com/sonarqube-cloud](https://docs.sonarsource.com/sonarqube-cloud/) — ainsi que
sur la configuration réelle du projet `spring-petclinic-rest` utilisé dans ce cours.

---

## 1️⃣ Qu'est-ce que l'analyse statique ?

Jusqu'ici, tout ce que vous avez automatisé pour évaluer la qualité du code **exécutait** le code
(les tests, semaine 4). L'analyse statique fait l'inverse : elle **lit** le code source — sans le
lancer — à la recherche de motifs connus pour être problématiques (complexité excessive,
duplication, mauvaise gestion des ressources, failles de sécurité classiques, etc.).

SonarQube analyse le code à l'aide de **règles**. La documentation officielle les décrit ainsi :

> « Dans SonarQube Cloud, les analyseurs fournissent des règles qui sont exécutées sur le code
> source pour générer des *issues*. [...] Chaque règle est associée à un **type** : bug,
> vulnérabilité, *code smell*, ou *security hotspot*. »
> — [SonarQube rules](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-rules/rules)

Chaque violation détectée dans votre code est appelée une **issue** (anomalie) :

> « Les *issues* SonarQube Cloud sont des erreurs potentielles trouvées dans votre base de code.
> [...] Une analyse détecte une *issue* comme un problème dans votre code. Lorsqu'une règle de
> codage est enfreinte, une *issue* est soulevée. »
> — [Issues — Introduction](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/issues/introduction)

---

## 2️⃣ Les trois qualités logicielles évaluées

SonarQube ne se contente pas de compter des anomalies : chaque règle est rattachée à l'une des
trois **qualités logicielles** suivantes, selon la documentation officielle :

> « Un code de haute qualité contribue à un logiciel sécurisé, fiable et maintenable. [...]
> - **Sécurité** est la protection de votre logiciel contre un accès, un usage ou une destruction
>   non autorisés.
> - **Fiabilité** est une mesure de la capacité de votre logiciel à maintenir son niveau de
>   performance dans des conditions données, pour une période donnée.
> - **Maintenabilité** fait référence à la facilité avec laquelle vous pouvez réparer, améliorer et
>   comprendre le code. »
> — [SonarQube rules — Software qualities](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-rules/rules)

| Qualité | Question posée par la règle | Type historique associé |
|---|---|---|
| **Sécurité** | Ce code pourrait-il être exploité par un attaquant ? | Vulnérabilité |
| **Fiabilité** | Ce code est-il manifestement erroné, ou plus probablement erroné que non ? | Bug |
| **Maintenabilité** | Ce n'est ni un bug, ni une vulnérabilité, mais... | *Code smell* |

> 💡 La documentation précise que les quatre **types** historiques (bug, vulnérabilité, *code
> smell*, *security hotspot*) sont **dépréciés** au profit des qualités logicielles ci-dessus :
> « Les types et les sévérités de type sont dépréciés. Utilisez les qualités logicielles et leurs
> sévérités pour classer et prioriser vos *issues*. » Les deux vocabulaires coexistent cependant
> dans l'interface actuelle de SonarCloud — vous verrez les deux.

Parmi ces quatre types, le ***security hotspot*** mérite une mention à part : ce n'est pas une
anomalie confirmée, mais du code **sensible** à la sécurité qui nécessite une **revue humaine**
avant d'être qualifié de vulnérabilité réelle ou de faux positif.

**Exemples concrets, un par qualité logicielle** (le genre de code que SonarQube signalerait
réellement dans un projet Java comme celui du cours) :

```java
// 🔴 Sécurité — injection SQL (requête construite par concaténation d'une entrée utilisateur)
String query = "SELECT * FROM owners WHERE id = " + request.getParameter("ownerId");
statement.executeQuery(query);
```

```java
// 🟠 Fiabilité — division par une variable jamais vérifiée (NullPointerException/ArithmeticException possible)
public double tauxReussite(int reussites, int total) {
    return reussites / total; // total == 0 fera planter l'appel, pas de garde
}
```

```java
// 🟡 Maintenabilité — code smell : bloc dupliqué presque à l'identique à trois endroits du projet
if (pet.getType().equals("dog")) {
    log.info("Traitement spécial chien");
    // ... 12 lignes quasi identiques répétées ailleurs dans PetController, VetController...
}
```

Le premier exemple est une **vulnérabilité** exploitable par un attaquant (qualité : sécurité); le
second est un **bug** potentiel, pas malveillant mais incorrect (qualité : fiabilité); le troisième
n'est ni l'un ni l'autre — le code *fonctionne*, mais sera coûteux à maintenir (qualité :
maintenabilité), exactement la distinction faite dans le tableau ci-dessus.

### Sévérités

Chaque qualité logicielle impactée par une règle reçoit une sévérité :

| Sévérité | Définition officielle (traduite) |
|---|---|
| **Blocker** | Probabilité significative de conséquences graves, à corriger immédiatement (ex. plantage en production, faille permettant l'extraction de données sensibles). |
| **High** | Impact élevé sur l'application, à corriger dès que possible. |
| **Medium** | Impact moyen sur l'application. |
| **Low** | Impact faible sur l'application. |

— [SonarQube rules — Severities](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-rules/rules)

**En reprenant les trois exemples ci-dessus** :

- L'injection SQL reçoit typiquement la sévérité **Blocker** en sécurité : une donnée contrôlée
  par l'utilisateur peut directement altérer la requête exécutée contre la base de données — le
  pire des scénarios couverts par la définition officielle.
- La division non protégée reçoit typiquement **High** en fiabilité : l'application plante dans un
  cas précis et reproductible, mais seulement si `total` vaut zéro — un impact réel, mais plus
  circonscrit qu'une faille de sécurité exploitable à distance.
- Le bloc dupliqué reçoit typiquement **Low** en maintenabilité : le code reste fonctionnel
  aujourd'hui, le coût se paie plus tard, au moment où il faudra corriger les trois copies au lieu
  d'une seule.

---

## 3️⃣ Quality Profiles : quelles règles sont réellement actives

Toutes les règles décrites dans les sections précédentes ne sont pas automatiquement appliquées
à votre projet. C'est un **Quality Profile** (profil de qualité) qui détermine, pour chaque
langage, l'ensemble précis de règles activées lors de l'analyse :

> « Un profil de qualité définit l'ensemble des règles de codage à appliquer durant l'analyse.
> [...] Chaque projet a un seul profil de qualité activé, pour chaque langage, à un moment donné.
> Par défaut, le profil de qualité intégré **Sonar way core** est activé pour chaque langage. »
> — [Quality profiles — Introduction](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-quality-profiles/introduction)

SonarQube Cloud fournit trois profils intégrés, chacun avec un niveau de rigueur différent :

> « **Sonar way core** active : toutes les règles de sécurité; les règles de fiabilité
> recommandées de sévérité moyenne ou plus; les règles de maintenabilité recommandées de sévérité
> *Blocker*. [...] **Sonar way extended** [...] ajoute de la rigueur en fiabilité et en
> maintenabilité. [...] **Sonar way comprehensive** active : toutes les règles de sécurité; toutes
> les règles de fiabilité recommandées; toutes les règles de maintenabilité recommandées. »
> — [Built-in quality profiles](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-quality-profiles/built-in-quality-profiles)

| Profil intégré | Rigueur | Cas d'usage typique |
|---|---|---|
| **Sonar way core** (par défaut) | La plus légère | Bon point de départ pour la plupart des projets |
| **Sonar way extended** | Intermédiaire | Équipe déjà à l'aise avec SonarQube, qui veut plus de rigueur |
| **Sonar way comprehensive** | Maximale | Équipe expérimentée, ou évaluation complète des capacités de l'outil |

> 💡 Les profils intégrés s'alignent aussi sur des référentiels de sécurité externes reconnus
> (OWASP Top 10, CWE Top 25, OWASP ASVS, entre autres) — une règle SonarQube n'est donc pas un
> critère arbitraire inventé par l'outil, mais traçable jusqu'à des standards de l'industrie.

**Exemple concret** : le projet `spring-petclinic-rest` du cours est analysé en Java. Par défaut,
c'est donc le profil **Sonar way core (Java)** qui est actif — visible dans SonarCloud sous
*Project Settings* → *Quality Profile*. Si une équipe bascule plutôt vers **Sonar way extended**,
des règles de fiabilité auparavant ignorées (sévérité *Low*) deviennent actives : du code qui
passait inaperçu jusque-là peut soudainement générer de nouvelles *issues*, et potentiellement
faire échouer un quality gate qui passait auparavant — sans qu'une seule ligne de code n'ait
changé.

---

## 4️⃣ Dette technique et notation de maintenabilité

Chaque *code smell* détecté n'a pas le même coût de correction. SonarQube chiffre ce coût sous
forme de **dette technique** :

> « La [dette technique](https://www.sonarsource.com/learn/technical-debt/) est la somme des coûts
> de remédiation des *issues* de maintenabilité. Le coût de remédiation d'une *issue* est l'effort
> (en minutes) évalué pour corriger l'*issue*. [...] Une journée de 8 heures est présumée lorsque la
> dette technique est affichée en jours. »
> — [Understanding measures and metrics — Technical debt](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)

Ce chiffre brut (en minutes ou en jours) devient plus parlant une fois rapporté à la taille du
projet, sous forme de **ratio de dette technique** :

> « Le ratio de dette technique est le rapport entre le coût de développement du logiciel et la
> dette technique (le coût pour la corriger). [...] `sqale_debt_ratio` = dette technique / (coût de
> développement d'une ligne de code × nombre de lignes de code), où le coût de développement d'une
> ligne de code est prédéfini (par défaut, 30 minutes). »
> — [Understanding measures and metrics — Technical debt ratio](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)

**Exemple concret donné par la documentation officielle** : pour un projet avec une dette
technique de **122 563 minutes** et **63 987 lignes de code**, le ratio est de
`122 563 / (30 × 63 987) = 6,4 %`. Selon le barème ci-dessous, ce projet obtiendrait une
notation de maintenabilité **B**.

Ce ratio est ensuite traduit en une note de **A à E**, affichée directement sur votre projet :

> « La notation de maintenabilité reflète la densité de dette technique relative à la taille du
> code. Un ratio plus bas donne une meilleure note. [...] **A** ≤ 5 % · **B** ≥ 5 % à < 10 % ·
> **C** ≥ 10 % à < 20 % · **D** ≥ 20 % à < 50 % · **E** ≥ 50 %. »
> — [Understanding measures and metrics — Maintainability rating](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)

> ⚠️ Une note **globale** flatteuse peut masquer un problème récent : exactement comme pour la
> couverture JaCoCo (section 8️⃣), SonarQube calcule **deux** valeurs distinctes pour la plupart de
> ses métriques — une pour le code global, une pour le « nouveau code » — précisément parce qu'un
> projet legacy avec une dette ancienne importante ne doit pas servir d'excuse pour accepter de la
> **nouvelle** dette.

---

## 5️⃣ Le quality gate et « Clean as You Code »

La **quality gate** (porte de qualité) est l'indicateur central de SonarCloud :

> « Une quality gate SonarQube Cloud est un indicateur qui vous indique si votre code atteint le
> niveau minimal de qualité requis pour votre projet. Elle consiste en un ensemble de conditions
> appliquées aux résultats de chaque analyse. Si les résultats de l'analyse atteignent ou dépassent
> les conditions de la quality gate, un statut **Passed** s'affiche; sinon, un statut **Failed**
> s'affiche. »
> — [Quality gates](https://docs.sonarsource.com/sonarqube-cloud/standards/quality-gates)

Le rôle concret d'une quality gate, selon la documentation, dépend du contexte :

> « Pour la branche principale et les autres branches de longue durée, la quality gate répond à la
> question : *Puis-je livrer mon code aujourd'hui ?* Pour les Pull Requests (et branches de courte
> durée), la quality gate répond à la question : *Puis-je fusionner cette Pull Request ?* »
> — [Quality gates — How quality gates are used](https://docs.sonarsource.com/sonarqube-cloud/standards/quality-gates)

> ⚠️ Une quality gate n'est **pas calculée** après une toute première analyse — il en faut au
> moins deux pour établir une comparaison avec le « nouveau code ». C'est normal de voir
> **Not computed** lors de la toute première exécution sur un projet.

### Le nouveau code : la logique derrière « Clean as You Code »

SonarSource résume sa philosophie par le slogan **« Clean as You Code »** : plutôt que d'exiger
une remise à niveau immédiate de tout un projet legacy, on se concentre sur le code **qu'on
touche aujourd'hui**. Concrètement, SonarQube calcule la plupart de ses métriques deux fois :

> « Pour la plupart des métriques, SonarQube calcule deux valeurs ou notations : une pour le code
> global et une pour le nouveau code. Le code global est tout le code analysé du projet ou de la
> branche, l'ancien code plus le nouveau code. Le nouveau code est le code qui correspond à votre
> définition du nouveau code. C'est le code qui a été récemment ajouté ou modifié. [...] Les
> valeurs calculées peuvent différer entre le code global et le nouveau code parce que la portée
> est différente. Par exemple : votre code global peut obtenir un A en notation de
> maintenabilité, tandis que le nouveau code introduit plus de dette technique et obtient un B. »
> — [Understanding measures and metrics](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)

Cette « définition du nouveau code » n'est pas figée : la documentation propose plusieurs façons
de la configurer par projet — par exemple *Number of days* (tout code modifié dans les X derniers
jours, 30 jours par défaut) ou *Previous version* (tout code modifié depuis le dernier
incrément de version) :

> « Tout code qui a changé dans les X derniers jours est considéré comme du nouveau code. [...] La
> valeur par défaut est 30 jours, 7 ou 14 jours sont d'autres valeurs courantes. »
> — [Quality standards and new code](https://docs.sonarsource.com/sonarqube-cloud/standards/about-new-code)

C'est précisément ce mécanisme qui permet à la quality gate par défaut de **ne cibler que le
nouveau code** : un projet legacy truffé de dette technique ancienne peut donc tout de même
obtenir un quality gate **Passed** sur ses dernières Pull Requests, tant que le code qui y est
ajouté respecte, lui, les standards de qualité — l'inverse (projet globalement propre mais
nouveau code bâclé) provoque, lui, un échec, comme illustré dans l'encadré « Testez-vous » plus
bas.

---

## 6️⃣ Intégration réelle au projet du cours

Le dépôt upstream `spring-petclinic-rest` déclenche une analyse SonarCloud directement depuis
Maven, dans son pipeline GitHub Actions (vu en référence à la semaine 5, déclenchement manuel dans
le dépôt du cours) :

```yaml
# .github/workflows/maven-build-master.yml (projet spring-petclinic-rest)
- name: Build and analyze
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: ./mvnw -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=spring-petclinic_spring-petclinic-rest -Dsonar.organization=spring-petclinic
```

Deux éléments essentiels dans cette commande :

- Le **plugin Maven du scanner SonarQube** (`sonar-maven-plugin:sonar`) s'ajoute directement après
  `verify` — l'analyse ne remplace pas le cycle de vie Maven habituel (semaine 3), elle s'y greffe.
- Un **jeton d'authentification** (`SONAR_TOKEN`) est requis en plus du `GITHUB_TOKEN` habituel —
  ce jeton est généré depuis SonarCloud et stocké comme secret du dépôt GitHub, exactement comme
  vous l'avez fait pour d'autres secrets à la semaine 5.

Le `pom.xml` du même projet déclare les propriétés correspondantes :

```xml
<!-- SonarQube configuration -->
<sonar.organization>spring-petclinic-rest</sonar.organization>
<sonar.host.url>https://sonarcloud.io</sonar.host.url>
```

> ⚠️ Notez que la commande du workflow utilise `-Dsonar.organization=spring-petclinic` (sans
> `-rest`) tandis que le `pom.xml` déclare `spring-petclinic-rest`. Un paramètre passé en ligne de
> commande (`-D...`) **écrase** toujours la valeur équivalente définie dans le `pom.xml` — c'est la
> valeur de la commande qui est réellement utilisée à l'exécution. Ce genre d'écart, laissé sans
> conséquence ici parce qu'un seul des deux est actif, illustre bien pourquoi une configuration
> dupliquée à deux endroits est elle-même un *code smell* à surveiller.

---

## 7️⃣ Décoration des Pull Requests

SonarCloud n'attend pas qu'une Pull Request soit fusionnée pour l'analyser. C'est même l'un des
trois points d'intégration mis en avant par la documentation :

> « SonarQube Cloud s'intègre à votre flux de travail à trois endroits : dans l'IDE (via SonarQube
> for IDE), sur la Pull Request, et sur la base de code. »
> — [What SonarQube Cloud can do](https://docs.sonarsource.com/sonarqube-cloud/discovering-sonarcloud/what-sonarcloud-can-do)

Concrètement, l'analyse d'une Pull Request ne recalcule pas le quality gate du projet au complet :

> « L'analyse de Pull Request ne rapporte que les *issues* introduites par la Pull Request
> elle-même. Lorsque SonarQube Cloud analyse une Pull Request **P** pour la fusion de la branche
> **B** dans la branche de base **T**, il scanne le commit HEAD de **B** et compare le résultat
> avec la dernière analyse de **T**. Seules les *issues* qui apparaissent dans **B** mais pas dans
> **T** sont rapportées dans les résultats de l'analyse. »
> — [Pull request analysis](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/pull-request-analysis)

Le résultat (statut du quality gate et résumé des *issues*) ne reste pas confiné à l'interface de
SonarCloud : il est aussi publié directement dans la Pull Request GitHub elle-même, un mécanisme
appelé **décoration de Pull Request** :

> « En plus d'apparaître dans l'interface de SonarQube Cloud, le statut du quality gate et un
> résumé des résultats apparaissent aussi dans l'interface de votre plateforme DevOps (c'est-à-dire
> dans la vue de Pull Request de GitHub, Bitbucket Cloud, Azure DevOps ou GitLab). C'est ce qu'on
> appelle la décoration de Pull Request. Mais ce n'est pas que décoratif ! Elle s'intègre aussi à
> votre plateforme DevOps pour **bloquer la fusion** de la Pull Request si le quality gate
> échoue. »
> — [Pull request analysis — Pull request decoration](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/pull-request-analysis)

> ⚠️ Dans le plan gratuit de SonarQube Cloud, la documentation précise une limite importante pour
> des dépôts publics comme ceux du cours : « l'analyse de Pull Request n'est disponible que
> lorsque la Pull Request est fusionnée dans la branche principale ». La décoration **avant**
> fusion (commentaire en direct sur une PR encore ouverte) nécessite un plan payant — un détail à
> garder en tête si votre analyse n'apparaît pas tout de suite sur une PR encore ouverte.

**Exemple concret** : sur une Pull Request qui ajoute une méthode avec une duplication de code et
une couverture de tests insuffisante, le commentaire automatique déposé par SonarCloud affiche
typiquement :

```
Quality Gate Failed  ❌

2 New issues  •  0 Security Hotspots
Coverage on New Code: 42.0%  (required ≥ 80%)
Duplication on New Code: 15.0%  (required ≤ 3%)
```

Ce résumé (statut du quality gate, puis les métriques clés calculées uniquement sur le **nouveau
code** de la PR) permet à l'équipe de voir, directement dans l'onglet *Files changed* de GitHub,
pourquoi la fusion est bloquée — sans avoir à ouvrir SonarCloud.

Ce mécanisme explique pourquoi la section précédente distinguait deux questions différentes posées
par le quality gate : sur la branche principale, « puis-je livrer mon code aujourd'hui ? »; sur une
Pull Request, « puis-je fusionner cette Pull Request ? ». La décoration de Pull Request est
précisément ce qui rend cette seconde question visible **sans quitter GitHub**.

---

## 8️⃣ Fournir la couverture de tests à SonarCloud : JaCoCo

SonarCloud ne mesure pas lui-même la couverture de tests (semaine 4) : il **lit** un rapport déjà
produit par un autre outil, **JaCoCo**, et l'intègre à son analyse. Le `pom.xml` du projet
configure ce plugin avec deux exécutions Maven distinctes :

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${maven.jacoco-plugin.version}</version>
    <configuration>
        <excludes>
            <!-- Generated code -->
            <exclude>**/org/springframework/samples/petclinic/rest/dto/**</exclude>
            <exclude>**/org/springframework/samples/petclinic/rest/api/**</exclude>
        </excludes>
    </configuration>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.85</minimum>
                            </limit>
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.66</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
        <execution>
            <id>report</id>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

| Exécution | Rôle |
|---|---|
| `prepare-agent` | Instrumente le code **avant** l'exécution des tests, pour observer quelles lignes sont réellement exercées |
| `check` | Échoue le build Maven lui-même si la couverture globale (`BUNDLE`) descend sous **85 % des lignes** ou **66 % des branches** |
| `report` | Génère le rapport au format attendu par SonarCloud, **après** les tests |

> 💡 Les exclusions (`exclude`) retirent volontairement le code **généré automatiquement** (DTO et
> API générés par OpenAPI Generator, voir semaine 8) du calcul de couverture — ce code n'a jamais
> été écrit à la main, il serait absurde d'exiger qu'il soit testé comme du code métier.

Cette double configuration illustre une idée importante : un **quality gate** peut combiner des
métriques de **plusieurs outils** (ici, les règles SonarQube elles-mêmes **et** la couverture
JaCoCo) pour décider si un changement est acceptable.

<details>
<summary>🤔 Testez-vous</summary>

Une Pull Request ajoute une nouvelle méthode non testée dans une classe existante. Le reste du
projet est entièrement couvert. SonarCloud affiche un quality gate **Failed** sur cette PR
précisément à cause du « nouveau code ». Pourquoi le quality gate peut-il échouer **même si** la
couverture globale du projet reste au-dessus de 85 % ?

**Réponse** : la quality gate par défaut de SonarCloud applique ses conditions non seulement au
code existant, mais surtout au **nouveau code** introduit par la PR (« New Code »). Une méthode
neuve, non testée, peut donc faire échouer la quality gate sur cette PR précise, même si elle ne
fait presque pas bouger la moyenne globale du projet — l'objectif est justement d'empêcher que de
la nouvelle dette technique s'accumule, peu importe l'état historique du reste du code.
</details>

---

## 📚 Références

- [SonarQube rules](https://docs.sonarsource.com/sonarqube-cloud/standards/managing-rules/rules)
  — types de règles, qualités logicielles, sévérités.
- [Issues — Introduction](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/issues/introduction)
  — cycle de vie d'une *issue*.
- [Understanding measures and metrics](https://docs.sonarsource.com/sonarqube-cloud/managing-your-projects/metric-definitions)
  — dette technique, ratio de dette technique, notation de maintenabilité, code global vs nouveau code.
- [Quality gates](https://docs.sonarsource.com/sonarqube-cloud/standards/quality-gates)
  — définition et usage du quality gate.
- [Quality standards and new code](https://docs.sonarsource.com/sonarqube-cloud/standards/about-new-code)
  — définitions du « nouveau code » (Clean as You Code).
- Projet `spring-petclinic-rest` —
  [`.github/workflows/maven-build-master.yml`](https://github.com/spring-petclinic/spring-petclinic-rest/blob/main/.github/workflows/maven-build-master.yml)
  et [`pom.xml`](https://github.com/spring-petclinic/spring-petclinic-rest/blob/main/pom.xml).

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec [Analyse de vulnérabilités avec Trivy]({{% relref "/semaine-9/analyse-vulnerabilites-trivy" %}})
pour voir comment une deuxième famille d'outils détecte un risque que SonarQube ne couvre pas :
les failles **connues** dans vos dépendances et votre image Docker.
{{% /notice %}}
