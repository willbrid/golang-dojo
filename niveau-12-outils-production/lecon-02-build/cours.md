# Leçon 2 — Build de production

> Un programme Go se livre sous la forme d'un **seul fichier exécutable** : le *runtime*, la bibliothèque
> standard et toutes les dépendances y sont compilés. Il n'y a ni interpréteur ni machine virtuelle à
> installer, et on peut produire, depuis un poste Linux, les binaires de Windows, de macOS ou d'un serveur
> ARM. Cette leçon montre comment construire ce binaire pour la production : pour quelle plateforme, avec
> quelles informations, de quelle taille, et de façon **reproductible**.

## Objectifs

- Cross-compiler avec `GOOS` et `GOARCH`, et comprendre le rôle de `CGO_ENABLED`.
- Écrire du code spécifique à une plateforme ou à une option de build (contraintes de build, `-tags`).
- Injecter des valeurs dans le binaire avec `-ldflags -X`, en connaissant ses pièges.
- Réduire la taille du binaire (`-s -w`) en sachant ce qu'on perd et ce qu'on garde.
- Produire des builds **reproductibles** (`-trimpath`), et publier des sommes de contrôle.

## Explication

### Ce que contient un binaire Go

`go build` produit un exécutable qui contient le *runtime* (ordonnanceur, GC), la bibliothèque standard et
les dépendances. Il ne dépend que du noyau du système, et **éventuellement** de la bibliothèque C du système
(la *libc*) :

- **Avec cgo actif** (`CGO_ENABLED=1`, valeur par défaut quand un compilateur C est installé), certains
  paquets passent par la libc. `net` l'utilise pour la résolution des noms, `os/user` pour les comptes. Le
  binaire est alors **lié dynamiquement** : il ne démarre que sur un système qui a une libc compatible.
- **Avec `CGO_ENABLED=0`**, ces paquets utilisent leur implémentation en Go pur, et le binaire est
  **statique** : il tourne sur n'importe quel Linux de la même architecture, et même dans une image vide
  (leçon 3).

L'exemple le mesure : le même serveur HTTP, construit par défaut, dépend de `libc.so.6` ; construit avec
`CGO_ENABLED=0`, il ne dépend de rien. Pour un service, `CGO_ENABLED=0` est le choix par défaut. Un code qui a
réellement besoin de C (une bibliothèque C appelée par cgo) est une exception qu'on assume explicitement.

### La cross-compilation

Deux variables d'environnement choisissent la cible :

```
GOOS=linux   GOARCH=arm64  go build ./cmd/svc     # serveur ARM, Raspberry Pi 64 bits
GOOS=windows GOARCH=amd64  go build ./cmd/svc     # Windows : produit un .exe
GOOS=darwin  GOARCH=arm64  go build ./cmd/svc     # macOS sur puce Apple
```

`go tool dist list` affiche les combinaisons possibles (47 avec go1.27.3, dont 13 pour Linux). Rien d'autre
n'est nécessaire : pas de chaîne de compilation croisée à installer, tant que le code n'utilise pas cgo.
Quand on cross-compile, cgo est d'ailleurs **désactivé** par défaut.

Pour amd64, `GOAMD64=v1` (défaut), `v2`, `v3` ou `v4` autorise le compilateur à utiliser des instructions plus
récentes. `v3` exige notamment les instructions AVX2, présentes sur la plupart des processeurs x86 récents. Le binaire est un peu plus rapide, mais il
refuse de démarrer sur un processeur plus ancien. On ne change ce réglage que si un benchmark le justifie
(niveau 11), et en connaissant les machines cibles.

### Les contraintes de build

Un fichier peut n'être compilé que pour certaines plateformes ou certaines options. C'est nouveau dans le
cours : jusqu'ici, tous les fichiers d'un paquet étaient toujours compilés. Il y a deux mécanismes.

**Le nom du fichier.** Un suffixe `_GOOS`, `_GOARCH` ou `_GOOS_GOARCH` limite le fichier à cette cible :
`signal_windows.go`, `fast_amd64.go`, `io_linux_arm64.go`. Il en va de même pour les tests (`x_linux_test.go`).

**La ligne `//go:build`.** Placée **avant** la clause `package` (par convention suivie d'une ligne vide), elle donne une
expression booléenne sur des étiquettes :

```go
//go:build linux && !arm

//go:build debug

//go:build integration
```

- **Les étiquettes connues** sont la plateforme (`linux`, `windows`, `amd64`…), `cgo`, la version
  (`go1.27` : la version de Go est au moins 1.27), et `unix`.
- **Les étiquettes personnelles** sont activées par `go build -tags debug` (ou `-tags debug,trace`).
- **Les opérateurs** sont `&&`, `||`, `!` et les parenthèses.

L'usage le plus courant est une **paire de fichiers** : `debug_on.go` (`//go:build debug`) et `debug_off.go`
(`//go:build !debug`) qui définissent le même nom. Une seule des deux versions est compilée, et l'autre
n'existe pas dans le binaire. Les tests d'intégration lents se rangent souvent derrière
`//go:build integration` et ne tournent qu'avec `go test -tags integration ./...` (leçon 4).

Le piège : un fichier exclu n'est **ni compilé, ni vérifié** par `go vet` dans les builds ordinaires. Une
erreur dans `debug_on.go` ne se voit qu'avec `-tags debug`. La CI doit donc construire **chaque**
combinaison d'étiquettes utilisée.

### `-ldflags -X` : injecter une valeur

```
go build -ldflags "-X main.version=v1.2.0 -X example.com/svc/internal/meta.Commit=abc123" ./cmd/svc
```

`-X chemin.Variable=valeur` remplace, au moment de l'édition des liens, la valeur d'une **variable de type
`string`**. Le chemin est le chemin complet du paquet, sauf pour `main`. Les règles mesurées dans les
exercices :

- **Une variable initialisée par une constante** (`var version = "dev"`) est bien remplacée.
- **Une variable initialisée par un appel** (`var v = strings.ToUpper("…")`) n'est **pas** remplacée : le
  code d'initialisation s'exécute au démarrage et écrase la valeur.
- **Une constante, ou un nom mal orthographié**, est ignoré **sans aucune erreur**.

D'où la règle : une variable `string` initialisée par une constante, et un test, ou au moins un affichage
au démarrage, qui vérifie qu'elle a bien été remplacée.

Depuis Go 1.24, la version et le commit sont déjà inscrits par l'outil `go` (leçon 1). On utilise donc `-X`
pour ce que Git ne sait pas : un numéro de build de la CI, un canal de publication, ou la version quand le
build se fait **sans** dépôt Git (un `docker build` qui ne copie pas `.git`, leçon 3).

### `-s -w` : la taille

`-ldflags "-s -w"` retire la table des symboles (`-s`) et les informations de débogage DWARF (`-w`). Dans
l'exemple, le binaire passe de 8,6 Mo à 5,8 Mo (−32 %). Ce qu'on **garde** : les traces de pile d'une
panique, avec noms de fonctions, fichiers et numéros de ligne. Le *runtime* a sa propre table pour cela,
indépendante de celles qu'on retire. Ce qu'on **perd** : l'essentiel du débogage. `dlv` (leçon 4 du niveau 13) pose encore un point d'arrêt sur
`main.main`, mais avertit « no debug info found » et ne sait plus afficher les variables. On perd aussi
l'analyse par les outils qui lisent DWARF.

On peut garder le binaire complet dans les artefacts de la CI et déployer la version réduite. Pour un
service, 3 Mo de moins pèsent peu face à une image de conteneur, mais le gain est réel pour un outil
qu'on télécharge.

### Builds reproductibles

Un build est **reproductible** si les mêmes sources donnent, octet pour octet, le même binaire, où qu'on le
construise. C'est ce qui permet de vérifier qu'un binaire publié vient bien du code annoncé. L'outil `go` est
conçu pour cela, à une exception près : il inscrit dans le binaire le **chemin absolu** des sources, visible
dans les traces de pile.

`-trimpath` remplace ces chemins par des chemins de module (`example.com/svc/cmd/svc/main.go`). L'exemple
construit le même code dans deux répertoires : sans `-trimpath`, les deux binaires diffèrent ; avec, ils sont
identiques. `-trimpath` évite aussi de publier la structure des répertoires de la machine de build
(`/home/jean/…`).

L'empreinte d'un binaire dépend aussi :

- de la **version de Go** : `toolchain` dans `go.mod` fixe celle qui construit (N2 L9) ;
- des **options** (`-ldflags`, `-tags`, `GOAMD64`…) ;
- des **informations Git** (leçon 1) : deux clones du même commit donnent le même binaire, deux commits
  différents non ;
- d'un éventuel **cgo** : la chaîne C doit alors être identique aussi.

On publie les binaires avec un fichier de sommes de contrôle (`SHA256SUMS`). Quiconque reconstruit à partir
de la même étiquette, avec la même version de Go, peut comparer. La leçon 8 ajoutera la **signature**.

### PGO

Si un fichier `default.pgo` se trouve dans le répertoire du paquet `main`, `go build` l'utilise
automatiquement (N11 L4). C'est un fichier de plus à valider dans le dépôt, et une entrée de plus pour la
reproductibilité.

## Exemple

Un service minimal. Une option de build `debug` change une constante, et le script `build.sh` construit une
publication pour trois plateformes.

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27
```

```go
// Fichier : cmd/svc/main.go
// Commande svc : un petit serveur, pour observer ce que produit go build.
package main

import (
	"fmt"
	"net/http"
	"os"
	"runtime"
)

// Remplacées au build par -ldflags "-X main.version=… -X main.date=…".
var (
	version = "dev"
	date    = "inconnue"
)

func main() {
	fmt.Printf("svc %s (construit le %s, %s, %s/%s, debug=%v)\n",
		version, date, runtime.Version(), runtime.GOOS, runtime.GOARCH, debug)
	if len(os.Args) > 1 && os.Args[1] == "serve" {
		http.ListenAndServe("127.0.0.1:8080", nil)
	}
}
```

```go
// Fichier : cmd/svc/debug_off.go
//go:build !debug

package main

const debug = false
```

```go
// Fichier : cmd/svc/debug_on.go
//go:build debug

package main

const debug = true
```

```bash
#!/usr/bin/env bash
# build.sh VERSION : construit svc pour trois plateformes, comme le ferait une publication.
set -euo pipefail
version=${1:?usage : build.sh VERSION}
date=2026-03-01 # en vrai : la date du commit, git log -1 --format=%cs

rm -rf dist && mkdir dist
for target in linux/amd64 linux/arm64 windows/amd64; do
	os=${target%/*} arch=${target#*/}
	out=dist/svc-$version-$os-$arch
	[ "$os" = windows ] && out=$out.exe
	CGO_ENABLED=0 GOOS=$os GOARCH=$arch go build \
		-trimpath \
		-ldflags "-s -w -X main.version=$version -X main.date=$date" \
		-o "$out" ./cmd/svc
done
(cd dist && sha256sum svc-* > SHA256SUMS)

ls -l dist | awk 'NR > 1 { printf "%9d  %s\n", $5, $9 }'
cut -c1-16 dist/SHA256SUMS | paste - <(cut -d' ' -f3 dist/SHA256SUMS)
dist/svc-$version-linux-amd64
```

Construire comme pendant le développement, puis avec l'étiquette `debug` :

```
$ go build -o bin/svc ./cmd/svc && bin/svc
svc dev (construit le inconnue, go1.27.3, linux/amd64, debug=false)
$ go build -tags debug -o bin/svc-debug ./cmd/svc && bin/svc-debug
svc dev (construit le inconnue, go1.27.3, linux/amd64, debug=true)
$ ldd bin/svc
	linux-vdso.so.1 (0x00007ffc06fd0000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x0000735285e00000)
	/lib64/ld-linux-x86-64.so.2 (0x0000735286180000)
```

La publication :

```
$ ./build.sh v1.2.0
      273  SHA256SUMS
  5787808  svc-v1.2.0-linux-amd64
  5374112  svc-v1.2.0-linux-arm64
  5979136  svc-v1.2.0-windows-amd64.exe
2a3380e58b732d8d	svc-v1.2.0-linux-amd64
1d3cef43c1ee2401	svc-v1.2.0-linux-arm64
42f428b4d143a8f7	svc-v1.2.0-windows-amd64.exe
svc v1.2.0 (construit le 2026-03-01, go1.27.3, linux/amd64, debug=false)
```

Le même script, lancé dans une **copie** du projet située dans un autre répertoire, donne les mêmes sommes
de contrôle (`2a3380e58b732d8d…`, `1d3cef43c1ee2401…`, `42f428b4d143a8f7…`). Avec go1.27.3, les mêmes
sources, hors d'un dépôt Git, tu dois obtenir les mêmes chez toi.

Les mesures sur le binaire Linux/amd64 :

| Construction | Taille | Liaison |
|---|---|---|
| `go build` | 8 626 483 octets | dynamique (libc) |
| `-ldflags "-s -w"` | 5 849 351 octets (−32 %) | dynamique |
| `-trimpath -ldflags "-s -w"`, `CGO_ENABLED=0` (`build.sh`) | 5 787 808 octets | **statique** |

| Construction dans deux répertoires différents | Empreintes |
|---|---|
| `go build` | `e0ad04ce…` et `b3710baa…` : différentes |
| `go build -trimpath` | `74929752…` et `74929752…` : identiques |

## Explication du code

| Élément | Rôle |
|---|---|
| `var version = "dev"` | Une variable `string` initialisée par une constante : la seule forme que `-X` sait remplacer. La valeur `dev` signale un build sans `-X`. |
| `//go:build !debug` / `//go:build debug` | Deux fichiers, une seule constante `debug` : exactement un des deux est compilé. Sans aucune des deux lignes, la constante serait définie deux fois et le build échouerait. |
| `CGO_ENABLED=0` | Binaire statique, identique quelle que soit la machine de build (pas de libc ni de compilateur C en jeu). |
| `-trimpath` | Chemins de module au lieu des chemins absolus : build reproductible, et aucune fuite de la structure de la machine de build. |
| `-ldflags "-s -w -X …"` | Toutes les options de l'éditeur de liens passent dans **une seule** chaîne `-ldflags`. Deux `-ldflags` successifs : seul le dernier compte. |
| `${target%/*}`, `${target#*/}` | Découpe `linux/arm64` en `linux` et `arm64` (syntaxe du shell). |
| `SHA256SUMS` | Les sommes publiées avec les binaires, au format de `sha256sum -c`. |
| `date=2026-03-01` | Une date fixe pour l'exemple. En vrai, la date **du commit** (`git log -1 --format=%cs`), pas l'heure du build, qui rendrait le build non reproductible. |

## Erreurs fréquentes

1. **Un binaire lié dynamiquement** copié sur une autre distribution ou dans une image minimale : `no such file
   or directory` au démarrage, alors que le fichier existe (c'est l'interpréteur `ld-linux` qui manque).
2. **`-X` sur une constante, un nom mal écrit, ou une variable initialisée par un appel** : aucun message,
   aucun effet.
3. **Deux `-ldflags`** sur la même ligne de commande : le second annule le premier.
4. **L'heure du build** injectée dans le binaire : chaque build est différent.
5. **Oublier `-trimpath`** : builds non reproductibles et chemins personnels publiés.
6. **Une étiquette de build jamais testée en CI** : le code derrière ne compile plus depuis des mois.
7. **Placer `//go:build` après la clause `package`** : le compilateur refuse (`misplaced compiler
   directive`). Elle doit être en tête du fichier, seulement précédée de commentaires et de lignes vides.

## Bonnes pratiques Go

- Pour un service : `CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o … ./cmd/…`.
- La version vient de Git (leçon 1) ; `-X` pour les compléments, sur des variables `string` initialisées par
  une constante, vérifiées au démarrage.
- Les cibles de cross-compilation listées dans un script ou dans la CI, jamais tapées à la main.
- Une étiquette de build = une combinaison testée en CI.
- `SHA256SUMS` publié avec les binaires ; version de Go fixée (`toolchain`).
- Garder les binaires non réduits dans les artefacts si l'on veut pouvoir les déboguer.

## Ce que je dois retenir

- Un binaire Go est autonome ; avec `CGO_ENABLED=0`, il est même statique.
- `GOOS`/`GOARCH` suffisent pour cross-compiler du Go pur.
- `//go:build` et les suffixes `_linux.go` sélectionnent des fichiers ; `-tags` active des étiquettes.
- `-X` ne remplace que des variables `string` initialisées par une constante, et n'avertit jamais.
- `-s -w` : −32 % ici, traces de pile conservées, débogueur perdu.
- `-trimpath` rend le build reproductible : mêmes sources, même Go, mêmes options → mêmes octets.

---

➡️ [Exercices](exercices.md)
