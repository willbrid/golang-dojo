# Leçon 9 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Anatomie

```
module github.com/willbrid/demo   ← chemin d'import racine de TOUS les packages du module

go 1.27                            ← version du LANGAGE demandée, pas du compilateur

require (
	github.com/google/uuid v1.6.0  ← dépendance DIRECTE : importée par le code
	golang.org/x/text v0.14.0      ← version MINIMALE requise, pas figée
)

require (
	github.com/kr/pretty v0.3.1 // indirect   ← amenée par une dépendance de dépendance
)
```

`go.sum`, deux lignes par module :

```
github.com/google/uuid v1.6.0 h1:NIvaJ…=          ← empreinte du CONTENU du module
github.com/google/uuid v1.6.0/go.mod h1:TIyP…=    ← empreinte de son go.mod SEUL
```

**Pourquoi deux ?** Go doit lire le `go.mod` de chaque dépendance pour construire le graphe et
appliquer MVS — **sans télécharger le code**. Il vérifie donc l'intégrité du `go.mod` seul avant
de décider s'il a besoin du reste. C'est ce qui rend la résolution de dépendances rapide et
sûre : on ne télécharge que ce qui sera effectivement utilisé.

Le préfixe `h1:` désigne l'algorithme de hachage utilisé (SHA-256 sur une arborescence).

## E2 — `go mod why`

```bash
$ go list -m all | wc -l
23                        # 2 dépendances directes, 21 indirectes

$ go mod why golang.org/x/sys
# golang.org/x/sys
github.com/willbrid/demo
github.com/charmbracelet/lipgloss
github.com/muesli/termenv
golang.org/x/sys
```

La sortie se lit **de haut en bas** : mon module importe `lipgloss`, qui importe `termenv`, qui
importe `x/sys`.

L'exercice révèle presque toujours la même chose : **deux dépendances directes en amènent
vingt**. C'est l'argument central de la culture Go du « peu de dépendances » — chaque `require`
ajoute une surface de maintenance, de sécurité et de compilation qu'on n'a pas choisie.

`go mod why -m <module>` donne la réponse au niveau du module plutôt que du package.

## E3 — Retirer une dépendance

Après suppression de l'import :
- le projet **compile** parfaitement ;
- `go.mod` est **inchangé** ;
- `go.sum` est inchangé.

Après `go mod tidy` :
```diff
-require github.com/google/uuid v1.6.0
```
et les lignes correspondantes disparaissent de `go.sum`.

**Le rôle exact de `go mod tidy`** : synchroniser `go.mod` et `go.sum` avec les imports
**réellement présents dans le code** — en ajoutant ce qui manque et en retirant ce qui ne sert
plus. Le compilateur, lui, se contente d'utiliser ce qui est déclaré ; il ne se plaint pas d'un
`require` superflu.

D'où la règle : **`go mod tidy` avant chaque commit**, comme `gofmt` et `go vet`. Un `go.mod`
qui diverge du code accumule des dépendances fantômes.

## E4 — Version du langage

Avec `go 1.21` :
```
333
```
Avec `go 1.27`, **sans rien changer d'autre au code** :
```
012
```

La ligne `go` du `go.mod` ne décrit pas une exigence d'installation : elle **sélectionne la
sémantique du langage**. Le même compilateur Go 1.27 applique l'ancienne règle de la variable
de boucle si le module déclare `go 1.21`.

C'est la promesse de compatibilité de Go rendue opérationnelle : un module écrit en 2023
continue de se comporter comme en 2023, même compilé en 2026. Et c'est aussi pourquoi mettre à
jour cette ligne est un **acte délibéré**, à faire en connaissance de cause.

## E5 — `go install` contre dépendance

```bash
$ go install golang.org/x/tools/cmd/goimports@latest
$ git diff go.mod           # aucune modification
$ which goimports
/home/willbrid/go/bin/goimports     # $GOPATH/bin
```

`go install pkg@version` construit et installe un **binaire**, dans un module éphémère isolé.
Il ne touche jamais le `go.mod` du projet courant.

Avec la directive `tool` (Go 1.24+) :
```
tool golang.org/x/tools/cmd/goimports
```
```bash
$ go tool goimports -w .
```

| | `go install …@version` | directive `tool` |
|---|---|---|
| Version | choisie par chaque développeur | **figée dans le dépôt**, identique pour tous |
| `go.mod` | intact | modifié |
| Reproductibilité en CI | à gérer à part | automatique |
| Usage | outil personnel (`delve`, `gopls`) | outil du projet (`mockgen`, `sqlc`, linters) |

La directive `tool` remplace le bricolage historique du fichier `tools.go` rempli d'imports
anonymes, qu'on croise encore dans beaucoup de projets.

## Exercice intermédiaire — publier un module

**Contrainte 5 — pourquoi `v0.1.0` et pas `v1.0.0` ?**

En semver, `v1.0.0` est une **promesse publique de stabilité** : à partir de là, toute rupture
de compatibilité exige un `v2` avec un chemin d'import différent. Les versions `v0.x.y` sont
explicitement hors garantie — tout peut casser d'une mineure à l'autre, et les utilisateurs le
savent.

Publier `v1.0.0` sur une API qu'on n'a jamais fait relire par personne, c'est s'engager
définitivement sur des choix qu'on n'a pas encore éprouvés. Et les versions Go étant
**immuables**, on ne peut pas revenir en arrière.

**Contrainte 7 — pourquoi le consommateur ne casse pas.**

```bash
# module consommateur, go.mod
require github.com/willbrid/slugify v0.1.0
```

Publier `v0.2.0` avec un renommage de fonction ne change **rien** pour le consommateur : MVS
sélectionne la version **déclarée**, pas la dernière disponible. Tant que personne n'écrit
`go get -u`, rien ne bouge.

C'est la différence fondamentale avec npm, où `^0.1.0` accepterait automatiquement `0.2.0`. En
Go, **une mise à jour est toujours un acte explicite**, visible dans un commit et attribuable.
C'est ce qui permet de se passer de fichier de verrouillage.

**Contrainte 8 — rompre la compatibilité après `v1.0.0`.**

Il faut publier un **`v2`**, ce qui suppose :
1. modifier la ligne `module` en `github.com/willbrid/slugify/v2` ;
2. étiqueter `v2.0.0` ;
3. les utilisateurs importent `github.com/willbrid/slugify/v2` — **un chemin différent**.

Deux stratégies pour la structure du dépôt :

| | Sous-répertoire `/v2` | Branche dédiée |
|---|---|---|
| Structure | `slugify/v2/` avec son propre `go.mod` | branche `v2`, racine inchangée |
| v1 et v2 maintenues ensemble | facile (même arbre) | difficile (deux branches à synchroniser) |
| Duplication de code | réelle | aucune |
| Lisibilité pour un nouveau venu | moyenne | bonne |

La branche dédiée est la voie recommandée par la documentation officielle quand v1 n'est plus
activement développée. Le sous-répertoire convient quand les deux versions doivent évoluer en
parallèle.

## Défi

**a) MVS à la main**

```
A → B v1.2.0, C v1.5.0
B v1.2.0 → D v1.1.0
C v1.5.0 → D v1.3.0, E v2.0.0
E v2.0.0 → D v1.2.0
```

Versions minimales demandées pour `D` : **1.1.0**, **1.3.0**, **1.2.0**.
MVS retient le **maximum des minimums** : **D v1.3.0**.

Résultat complet : `B v1.2.0`, `C v1.5.0`, `D v1.3.0`, `E v2.0.0`.

**Si A ajoute `D v1.0.0` en dépendance directe :** rien ne change. `1.0.0` s'ajoute à
l'ensemble des minimums, dont le maximum reste `1.3.0`. On ne peut pas **rétrograder** une
dépendance en baissant sa propre exigence — il faut agir sur celui qui demande la version
haute, ou utiliser `exclude`.

**Si E passait en `v3.0.0` :** le chemin d'import deviendrait `…/e/v3`. `C` continuerait
d'importer `…/e` (v2) tant qu'il n'est pas mis à jour ; **les deux versions coexisteraient**
dans le binaire, comme deux modules distincts. C'est précisément ce que permet la règle du
suffixe majeur.

**Comparaison avec npm.** npm résoudrait « la plus récente compatible », donc potentiellement
`D v1.9.3` sortie ce matin — un build reproductible seulement grâce au `package-lock.json`, et
dont le contenu change dès qu'on régénère le lock. MVS produit un résultat **déterministe à
partir des seuls `go.mod`**, identique aujourd'hui et dans cinq ans. En contrepartie, on ne
bénéficie pas automatiquement des correctifs : il faut lancer `go get -u`. C'est un arbitrage
« reproductibilité contre fraîcheur », et Go a choisi le premier terme.

**b) Migration en v2**

```bash
# Stratégie « branche dédiée »
git checkout -b v2
sed -i 's|^module github.com/willbrid/slugify$|module github.com/willbrid/slugify/v2|' go.mod
# … rupture d'API …
git commit -am "v2 : renommage de Slugify en Make"
git tag v2.0.0 && git push origin v2 --tags
```

```go
// Les deux versions dans le MÊME programme
import (
	slug1 "github.com/willbrid/slugify"
	slug2 "github.com/willbrid/slugify/v2"
)

fmt.Println(slug1.Slugify("Bonjour"))  // ancienne API
fmt.Println(slug2.Make("Bonjour"))     // nouvelle API
```

**Pourquoi c'est possible :** pour Go, `…/slugify` et `…/slugify/v2` sont **deux modules
différents**, avec deux chemins d'import distincts. Il n'y a aucun conflit à résoudre — pas
plus qu'entre deux bibliothèques sans rapport.

**Ce que cela change pour une grande base de code :** la migration devient **progressive**. On
peut convertir un package aujourd'hui, un autre le mois prochain, sans jamais casser le build.
Dans un écosystème où une seule version d'un paquet peut exister à la fois, une migration
majeure est un basculement atomique de tout le dépôt — ce qui, sur des millions de lignes,
n'arrive jamais et fige les versions pendant des années.

**c) Chaîne d'approvisionnement**

```bash
$ govulncheck ./...
Vulnerability #1: GO-2024-2687
    HTTP/2 CONTINUATION flood in net/http
  More info: https://pkg.go.dev/vuln/GO-2024-2687
  Standard library
    Found in: net/http@go1.21.0
    Fixed in: net/http@go1.21.9
```

`govulncheck` ne se contente pas de comparer des numéros de version : il analyse le **graphe
d'appels** et ne signale que les vulnérabilités dont le code fautif est réellement atteignable
depuis le programme. Le taux de faux positifs est donc bien plus bas que celui d'un scanner
classique.

**Comment Go se protège d'une substitution en amont :** par `go.sum` et la **base de sommes de
contrôle** (`sum.golang.org`), un journal transparent et infalsifiable (*transparency log*) qui
enregistre l'empreinte de chaque version publiée. Toute divergence entre le contenu téléchargé
et l'empreinte connue provoque une erreur de sécurité explicite, avant toute compilation.

**Si un auteur force la réécriture d'une étiquette git déjà publiée :** rien ne change pour
ceux qui l'ont déjà. Le proxy (`proxy.golang.org`) a mis en cache le contenu **pour toujours**,
et c'est lui qui sert les téléchargements. Un nouvel utilisateur obtiendra donc l'ancien
contenu ; si quelqu'un contourne le proxy et récupère la version réécrite, la vérification
`go.sum` **échouera bruyamment** :

```
SECURITY ERROR
This download does NOT match the one reported by the checksum server.
```

C'est exactement le scénario que ce mécanisme est conçu à détecter.

**Les trois variables :** `GOPRIVATE` et `GONOSUMDB` existent —
`GOPRIVATE` est la forme moderne et couvre à la fois `GONOSUMDB` et `GONOPROXY`.
**`GONOSUMCHECK` n'existe pas** : c'était une variable de `dep`, l'outil antérieur aux modules.
`go help environment` fait foi, et c'était le but de la question : vérifier plutôt que se fier
à ce qu'on croit avoir lu.

Pour un dépôt d'entreprise privé : `GOPRIVATE=github.com/masociete/*`, sinon le proxy public
tente d'atteindre un dépôt inaccessible et l'échec est incompréhensible.

## Réponses du quiz

1. Un **dépôt** est un espace de stockage ; un **module** est l'unité de versionnement et de
   distribution, défini par un `go.mod` ; un **package** est un répertoire compilé ensemble. En
   général : un dépôt = un module ⊃ plusieurs packages.
2. La version du **langage** demandée. Elle active les fonctionnalités correspondantes et fixe
   la sémantique (variable de boucle, etc.). Depuis Go 1.21, elle déclenche aussi le
   téléchargement automatique du toolchain adéquat si le local est plus ancien.
3. À garantir l'**intégrité** du code téléchargé. **Oui**, il se committe toujours.
4. L'une pour le **contenu** du module, l'autre pour son **`go.mod` seul** — que Go doit lire
   pour construire le graphe sans télécharger le code.
5. Que la dépendance n'est pas importée directement par le code du module, mais amenée par une
   autre dépendance.
6. Que le **numéro majeur figure dans le chemin d'import** : `github.com/user/lib/v2`, et dans
   la ligne `module` du `go.mod`.
7. **Oui** : ce sont deux modules distincts pour Go, sans conflit possible.
8. La **sélection de version minimale** : Go retient le maximum des versions minimales
   demandées. npm retient la plus récente compatible, ce qui rend le résultat dépendant de la
   date de résolution.
9. Parce que MVS rend le résultat **déterministe à partir des seuls `go.mod`** : le verrouillage
   est déjà dans la déclaration. `go.sum` garantit l'intégrité, pas la sélection.
10. Il ajoute les dépendances manquantes, retire les inutilisées et met `go.sum` à jour. À
    lancer avant chaque commit.
11. Parce qu'il redirige un chemin d'import vers un répertoire local qui n'existe **que sur la
    machine de l'auteur** : le module devient inutilisable pour tout le monde. Utiliser
    `go.work`, qui n'est pas versionné.
12. **Non.** Elle reste dans le cache du proxy pour toujours. Seul `retract` permet de signaler
    qu'elle ne doit plus être utilisée — sans la faire disparaître.
