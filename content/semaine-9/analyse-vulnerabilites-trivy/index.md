+++
title = "Analyse de vulnérabilités avec Trivy"
weight = 2
draft = false
+++

Tout ce qui suit est fondé sur la documentation officielle du projet open source **Trivy**
(Aqua Security) — voir le [dépôt GitHub](https://github.com/aquasecurity/trivy) et la
[documentation](https://trivy.dev/docs/latest/) — ainsi que sur le
[`README` officiel de l'action GitHub `trivy-action`](https://github.com/aquasecurity/trivy-action).

> 💡 Contrairement à SonarCloud (page précédente), Trivy est un outil **entièrement open source et
> gratuit**, sans compte, sans organisation et sans distinction entre dépôt public ou privé —
> aucune licence n'est requise, peu importe la visibilité de votre dépôt de TP.

---

## 1️⃣ Un risque que SonarQube ne couvre pas

SonarQube (page précédente) analyse le code **que vous avez écrit**. Mais une application Spring
Boot comme celle du cours dépend de **dizaines** de bibliothèques tierces (voir le `pom.xml`,
semaine 3) et tourne, une fois conteneurisée (semaine 7), à l'intérieur d'une image Docker qui
embarque elle-même un système d'exploitation complet. Une faille peut donc exister dans du code
que vous n'avez **jamais** écrit vous-même — et qui a pourtant été rendue publique.

Une telle faille documentée publiquement est un **CVE** (*Common Vulnerabilities and Exposures*) —
un identifiant standardisé (ex. `CVE-2024-12345`) permettant à n'importe quel outil de sécurité de
référencer précisément la même faille.

Trivy se décrit lui-même ainsi :

> « Trivy est un scanneur de sécurité complet et polyvalent. Trivy possède des *scanners* qui
> cherchent des problèmes de sécurité, et des *cibles* où il peut trouver ces problèmes. »
> — [README officiel, aquasecurity/trivy](https://github.com/aquasecurity/trivy)

| Cibles (*targets*) que Trivy peut analyser | Scanners (*ce qu'il y cherche*) |
|---|---|
| Image de conteneur | Paquets système et dépendances logicielles (SBOM) |
| Système de fichiers | Vulnérabilités connues (CVE) |
| Dépôt Git distant | Problèmes d'*Infrastructure as Code* et mauvaises configurations |
| Image de machine virtuelle | Informations sensibles et secrets |
| Cluster Kubernetes | Licences logicielles |

> 💡 Ce tableau explique pourquoi Trivy complète SonarQube plutôt que de le remplacer : SonarQube
> analyse la **logique de votre code source**; Trivy analyse les **composants dont votre
> application dépend** (dépendances Maven, image de base Docker) à la recherche de failles **déjà
> documentées ailleurs**, pas de défauts de conception.

---

## 2️⃣ Les niveaux de sévérité

Trivy classe chaque vulnérabilité trouvée selon une échelle de sévérité standard dans l'industrie
de la sécurité (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN`), dérivée du score CVSS attribué à
chaque CVE. En pratique, dans un pipeline CI, on ne fait généralement échouer le build que sur les
niveaux les plus sérieux — c'est exactement le choix fait dans l'exemple officiel ci-dessous
(`CRITICAL,HIGH` uniquement), pour éviter qu'une vulnérabilité mineure, sans correctif disponible,
bloque inutilement toute l'équipe.

---

## 3️⃣ Scanner une image Docker dans GitHub Actions

L'exemple officiel de l'action `trivy-action`, utilisé tel quel dans un pipeline CI :

```yaml
# Exemple officiel — aquasecurity/trivy-action, "Scan CI Pipeline"
- name: Build an image from Dockerfile
  run: docker build -t docker.io/my-organization/my-app:${{ github.sha }} .

- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: 'docker.io/my-organization/my-app:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    ignore-unfixed: true
    vuln-type: 'os,library'
    severity: 'CRITICAL,HIGH'
```

— [aquasecurity/trivy-action, README — Scan CI Pipeline](https://github.com/aquasecurity/trivy-action)

| Option | Rôle |
|---|---|
| `image-ref` | L'image Docker à analyser — ici, celle tout juste construite par l'étape précédente (même idée que la construction d'image vue à la semaine 7) |
| `exit-code: '1'` | Fait **échouer le job** (comme un test qui échoue) si des vulnérabilités du niveau ciblé sont trouvées — sans cette option, Trivy se contente d'afficher un rapport sans jamais faire échouer le pipeline |
| `ignore-unfixed: true` | Ignore les vulnérabilités pour lesquelles **aucun correctif n'existe encore** — inutile de bloquer l'équipe pour un problème qu'elle ne peut, de toute façon, pas corriger elle-même |
| `vuln-type: 'os,library'` | Analyse à la fois les paquets du système d'exploitation de l'image **et** les bibliothèques applicatives (ex. les dépendances Maven empaquetées) |
| `severity: 'CRITICAL,HIGH'` | Ne considère (et ne fait échouer le build) que sur les deux niveaux de sévérité les plus élevés |

> ⚠️ `exit-code: '1'` est ce qui transforme Trivy d'un simple outil de rapport en une **porte**
> bloquante du pipeline — exactement le même principe qu'un `./mvnw verify` qui échoue si un test
> échoue (semaine 4), ou qu'un quality gate SonarCloud en `Failed` (page précédente). Sans cette
> option, l'étape Trivy réussirait **toujours**, peu importe ce qu'elle trouve.

---

## 4️⃣ Publier les résultats dans l'onglet Security de GitHub (SARIF)

Plutôt qu'un simple tableau dans les logs, Trivy peut produire un rapport au format **SARIF**
(*Static Analysis Results Interchange Format*, un format standard partagé par de nombreux outils
d'analyse) et le publier directement dans l'onglet **Security** du dépôt GitHub :

```yaml
# Exemple officiel — aquasecurity/trivy-action, "Scan CI Pipeline (w/ SARIF)"
jobs:
  build:
    runs-on: ubuntu-24.04
    permissions:
      contents: read          # Required to checkout and read repo files
      security-events: write  # Required to upload SARIF files to Security tab
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build an image from Dockerfile
        run: |
          docker build -t docker.io/my-organization/my-app:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: 'docker.io/my-organization/my-app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

— [aquasecurity/trivy-action, README — Scan CI Pipeline (w/ SARIF)](https://github.com/aquasecurity/trivy-action)

Deux détails essentiels :

- Le bloc `permissions` doit explicitement accorder `security-events: write` — par défaut, un
  workflow GitHub Actions n'a **pas** la permission de publier dans l'onglet Security, même s'il a
  accès en lecture au reste du dépôt.
- `if: always()` fait en sorte que les résultats soient téléversés **même si** une étape
  précédente a échoué — utile si vous combinez cette étape avec un `exit-code: '1'` : vous voulez
  voir le détail des vulnérabilités dans l'onglet Security, pas seulement un pipeline rouge sans
  explication.

> 💡 Combiner les deux approches (`exit-code: '1'` **et** l'upload SARIF) est possible, et courant :
> une première étape Trivy en mode `sarif` (sans faire échouer le build) publie le rapport complet
> dans l'onglet Security, pendant qu'une seconde étape, en mode `table` avec `exit-code: '1'`,
> bloque réellement le pipeline. Chaque étape a un rôle distinct : **visibilité** pour la première,
> **porte bloquante** pour la seconde.

<details>
<summary>🤔 Testez-vous</summary>

Votre pipeline configure Trivy avec `severity: 'CRITICAL,HIGH'` et `exit-code: '1'`, mais
**sans** `ignore-unfixed`. Une vulnérabilité `CRITICAL` est détectée dans une bibliothèque système
de l'image de base, mais aucun correctif n'est encore disponible auprès du mainteneur. Que se
passe-t-il, et comment l'option `ignore-unfixed: true` changerait-elle ce comportement ?

**Réponse** : sans `ignore-unfixed`, le pipeline **échoue** (`exit-code: '1'`) même si l'équipe ne
peut rien faire immédiatement pour corriger cette vulnérabilité précise — il n'existe pas encore de
version corrigée à installer. Avec `ignore-unfixed: true`, Trivy exclurait cette vulnérabilité du
calcul de `exit-code`, puisqu'aucune action corrective n'est actuellement possible; seules les
vulnérabilités **avec un correctif disponible** (donc une action concrète possible : mettre à jour
une dépendance ou l'image de base) feraient échouer le pipeline.
</details>

---

## 📚 Références

- [README officiel, aquasecurity/trivy](https://github.com/aquasecurity/trivy) — description,
  cibles et scanners de Trivy.
- [README officiel, aquasecurity/trivy-action](https://github.com/aquasecurity/trivy-action) —
  exemples complets « Scan CI Pipeline » et « Scan CI Pipeline (w/ SARIF) ».
- [GitHub Docs — Uploading a SARIF file to GitHub](https://docs.github.com/code-security/code-scanning/integrating-with-code-scanning/uploading-a-sarif-file-to-github) —
  format SARIF et permissions requises.

{{% notice tip "🧪 À vous de jouer" %}}
Poursuivez avec l'[exercice Qualité et sécurité]({{% relref "/semaine-9/exercice-qualite-securite" %}})
pour interpréter de vrais rapports SonarCloud et Trivy, puis corriger les enjeux qu'ils révèlent.
{{% /notice %}}
