# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — D'autres plateformes

```
$ for t in linux/riscv64 freebsd/amd64 linux/386 js/wasm; do
>   CGO_ENABLED=0 GOOS=${t%/*} GOARCH=${t#*/} go build -trimpath -ldflags "-s -w" -o svc-${t%/*}-${t#*/} ./cmd/svc
> done
```

| Cible | Taille | `file` |
|---|---|---|
| `linux/amd64` (cours) | 5 787 808 | ELF 64-bit, x86-64, statically linked |
| `linux/riscv64` | 5 177 504 | ELF 64-bit, UCB RISC-V, statically linked |
| `freebsd/amd64` | 5 742 762 | ELF 64-bit, x86-64, (FreeBSD), statically linked |
| `linux/386` | 5 677 216 | ELF 32-bit, Intel 80386, statically linked |
| `js/wasm` | 9 683 691 | WebAssembly (wasm) binary module |

L'ordre n'est pas celui qu'on attendrait. Le binaire 32 bits n'est que 2 % plus petit que celui d'amd64, et
c'est RISC-V, une architecture 64 bits, qui donne le plus petit. La taille dépend surtout du **codage des
instructions** propre à chaque architecture, et de ce que le compilateur sait en faire, bien plus que de la
taille des pointeurs. Une conclusion qu'on ne pouvait pas deviner : on mesure.

`js/wasm` produit un **module WebAssembly**, qui s'exécute dans un navigateur ou dans Node.js, avec le fichier
d'accompagnement fourni par Go (`$(go env GOROOT)/lib/wasm/wasm_exec.js`). Sur cette machine :

```
$ PATH=$PATH:$(go env GOROOT)/lib/wasm go_js_wasm_exec svc-js-wasm
svc dev (construit le inconnue, go1.27.3, js/wasm, debug=false)
```

Le programme démarre. `http.ListenAndServe` n'a, lui, pas de sens dans un navigateur : le même code se
compile, mais toutes les fonctions ne sont pas utilisables partout. C'est ce que les contraintes de build
permettent de gérer.

## E2 — Les pièges de `-X`

```go
// Fichier : internal/meta/meta.go
// Package meta : trois façons de déclarer une valeur, face à -ldflags -X.
package meta

import "strings"

var Version = "dev"                     // initialisée par une constante
var Upper = strings.ToUpper("calculée") // initialisée par un appel
const Const = "constante"               // une constante
```

```go
// Fichier : cmd/meta/main.go
package main

import (
	"fmt"

	"example.com/svc/internal/meta"
)

func main() { fmt.Println(meta.Version, meta.Upper, meta.Const) }
```

```
$ go build -ldflags "-X example.com/svc/internal/meta.Version=v9 \
    -X example.com/svc/internal/meta.Upper=remplacée \
    -X example.com/svc/internal/meta.Const=x \
    -X example.com/svc/internal/meta.Verison=y" -o meta ./cmd/meta
$ echo $?
0
$ ./meta
v9 CALCULÉE constante
```

- **`Version`** est remplacée.
- **`Upper`** ne l'est pas : `-X` fixe la valeur initiale dans le binaire, puis le code d'initialisation du
  paquet s'exécute au démarrage et l'écrase avec le résultat de `strings.ToUpper`.
- **`Const`** ne l'est pas : une constante n'existe pas en mémoire, le compilateur l'a recopiée partout.
- **`Verison`**, mal orthographié, ne fait rien.
- **Aucun message, et un code de retour 0.**

La vérification automatique, à la fin de `build.sh`, consiste à exécuter le binaire natif et contrôler ce
qu'il affiche :

```bash
got=$(dist/svc-$version-linux-amd64)
case $got in
"svc $version "*) echo "$got" ;;
*) echo "version non injectée : $got" >&2; exit 1 ;;
esac
```

Avec une faute de frappe dans `-X main.verson=` :

```
version non injectée : svc dev (construit le 2026-03-01, go1.27.3, linux/amd64, debug=false)
```

La publication échoue (code 1) au lieu de livrer un binaire `dev`.

## E3 — Code par plateforme

```go
// notify_linux.go : le suffixe suffit, pas de //go:build
package main

import "fmt"

func notify() { fmt.Println("notification Linux") }
```

```go
// notify_other.go
//go:build !linux

package main

import "fmt"

func notify() { fmt.Println("notification générique") }
```

```go
// slow_test.go
//go:build integration

package main

import "testing"

func TestSlow(t *testing.T) { t.Log("test d'intégration lancé") }
```

```
$ go run .
notification Linux
$ GOOS=windows go build -o e3.exe . && GOOS=windows go vet .   # construit et vérifié pour Windows
$ go test -v ./...
?   	example.com/e3	[no test files]
$ go test -v -tags integration ./...
=== RUN   TestSlow
    slow_test.go:7: test d'intégration lancé
ok  	example.com/e3	0.004s
$ go list -f '{{.GoFiles}} {{.IgnoredGoFiles}}' .
[main.go notify_linux.go] [notify_other.go slow_test.go]
$ GOOS=windows go list -f '{{.GoFiles}} {{.IgnoredGoFiles}}' .
[main.go notify_other.go] [notify_linux.go slow_test.go]
```

`go list` répond exactement à la question « quels fichiers sont compilés pour cette cible et ces
étiquettes ? ». Sans `-tags integration`, `go test` dit même qu'il n'y a **aucun** fichier de test : le test
n'existe pas pour lui.

C'est le danger signalé dans le cours. Si la CI ne lance jamais `-tags integration`, ce test peut cesser de
compiler sans que personne ne le voie. De même, `GOOS=windows go vet .` est la seule façon de vérifier
`notify_other.go` depuis Linux.

## E4 — Ce que `-s -w` retire

```
$ go tool nm full | wc -l
8328
$ go tool nm sw
reading sw: no symbol section
```

La table des symboles disparaît entièrement : 8 328 symboles d'un côté, aucun de l'autre. La trace de pile d'une panique, elle, garde les noms de fonctions, les fichiers et les numéros de ligne (vérifié avec un programme qui panique, construit avec `-s -w`). Le
*runtime* utilise sa propre table, que `-s -w` ne touche pas.

```
$ dlv exec ./full      # puis : break main.main, continue
Breakpoint 1 set at 0x6255f3 for main.main() …/cmd/svc/main.go:17
> [Breakpoint 1] main.main() …/cmd/svc/main.go:17 (hits goroutine(1):1 total:1)
$ dlv exec ./sw
Warning: no debug info found, some functionality will be missing such as stack traces and variable evaluation.
Breakpoint 1 set at 0x6255e0 for main.main() …/cmd/svc/main.go:17
```

Le débogueur pose encore le point d'arrêt (il s'appuie lui aussi sur la table du *runtime*), mais il prévient
qu'il ne saura afficher ni les variables ni la pile. Pour déboguer un problème de production, on garde donc,
dans les artefacts de la CI, le binaire **non réduit** construit au même commit.

## E5 — Casser la reproductibilité

Empreinte du binaire `linux/amd64` (12 premiers caractères) :

| Modification | Empreinte | Légitime ? |
|---|---|---|
| référence (`build.sh v1.2.0`) | `2a3380e58b73` | — |
| `date=$(date +%s%N)` : l'heure du build, deux constructions | `911635678856`, puis `5f186e591976` | **défaut** : le code n'a pas changé |
| sans `-trimpath`, deux répertoires | `73084fa3df7b` et `2e0772a7dd0d` | **défaut** : le binaire dépend du chemin de la machine |
| `GOAMD64=v3` | `e9fe4e6c321c` | légitime : un autre jeu d'instructions, un autre programme |
| `build.sh v1.2.1` | `488e6cf3ff38` | légitime : la version fait partie du binaire |

D'autres causes légitimes : un autre commit Git (leçon 1), une autre version de Go (voir le défi), une autre
étiquette de build. La règle : l'empreinte ne doit dépendre **que** de ce qu'on a voulu mettre dans le binaire
(sources, version, options), jamais de l'**endroit** ni du **moment** du build.

---

## Exercice intermédiaire — Un outil de publication en Go

Le module déclare maintenant `golang.org/x/sync` (pour `errgroup`, N6 L9) :

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27

require golang.org/x/sync v0.23.0
```

```go
// Fichier : cmd/release/main.go
// Commande release : construit, vérifie et résume une publication de svc.
//
//	go run ./cmd/release v1.2.0
package main

import (
	"bytes"
	"context"
	"crypto/sha256"
	"debug/buildinfo"
	"errors"
	"fmt"
	"io"
	"log"
	"os"
	"os/exec"
	"path/filepath"
	"runtime"
	"slices"
	"strings"

	"golang.org/x/sync/errgroup"
)

type Target struct{ OS, Arch string }

var targets = []Target{{"linux", "amd64"}, {"linux", "arm64"}, {"windows", "amd64"}, {"darwin", "arm64"}}

func main() {
	if err := run(context.Background(), os.Args[1:], os.Stdout); err != nil {
		log.Fatal(err)
	}
}

func run(ctx context.Context, args []string, out io.Writer) error {
	if len(args) != 1 || !strings.HasPrefix(args[0], "v") {
		return errors.New("usage : release vX.Y.Z")
	}
	version := args[0]
	if err := clean(ctx); err != nil {
		return err
	}
	dist := "dist"
	if err := os.RemoveAll(dist); err != nil {
		return err
	}
	if err := os.MkdirAll(dist, 0o755); err != nil {
		return err
	}
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(4)
	files := make([]string, len(targets))
	for i, t := range targets {
		files[i] = filepath.Join(dist, name(version, t))
		g.Go(func() error {
			if err := build(ctx, version, t, files[i]); err != nil {
				return err
			}
			return Verify(files[i], version, t)
		})
	}
	if err := g.Wait(); err != nil {
		return err
	}
	sums, err := checksums(files)
	if err != nil {
		return err
	}
	if err := os.WriteFile(filepath.Join(dist, "SHA256SUMS"), []byte(sums), 0o644); err != nil {
		return err
	}
	fmt.Fprint(out, sums)
	return nil
}

func name(version string, t Target) string {
	n := fmt.Sprintf("svc-%s-%s-%s", version, t.OS, t.Arch)
	if t.OS == "windows" {
		n += ".exe"
	}
	return n
}

// clean refuse un arbre Git modifié : le binaire ne correspondrait à aucun commit.
func clean(ctx context.Context) error {
	out, err := exec.CommandContext(ctx, "git", "status", "--porcelain").Output()
	if err != nil {
		return fmt.Errorf("git status : %w (la publication se fait depuis un dépôt Git)", err)
	}
	if len(bytes.TrimSpace(out)) > 0 {
		return fmt.Errorf("arbre modifié, publication refusée :\n%s", out)
	}
	return nil
}

func build(ctx context.Context, version string, t Target, out string) error {
	cmd := exec.CommandContext(ctx, "go", "build", "-trimpath",
		"-ldflags", "-s -w -X main.version="+version, "-o", out, "./cmd/svc")
	cmd.Env = append(os.Environ(), "CGO_ENABLED=0", "GOOS="+t.OS, "GOARCH="+t.Arch)
	if b, err := cmd.CombinedOutput(); err != nil {
		return fmt.Errorf("build %s/%s : %w\n%s", t.OS, t.Arch, err, b)
	}
	return nil
}

// Verify contrôle un binaire sans l'exécuter.
func Verify(path, version string, t Target) error {
	info, err := buildinfo.ReadFile(path)
	if err != nil {
		return fmt.Errorf("%s : %w", path, err)
	}
	want := map[string]string{"-trimpath": "true", "CGO_ENABLED": "0", "GOOS": t.OS, "GOARCH": t.Arch}
	got := map[string]string{}
	for _, s := range info.Settings {
		got[s.Key] = s.Value
	}
	for k, v := range want {
		if got[k] != v {
			return fmt.Errorf("%s : %s = %q, attendu %q", path, k, got[k], v)
		}
	}
	// Avec -trimpath, -ldflags n'est PAS enregistré dans les informations de build, et chercher la
	// chaîne de version dans le fichier donne des faux positifs (« v1.0.0 » figure dans le chemin d'un
	// paquet de la bibliothèque standard). On exécute donc le binaire quand la plateforme le permet.
	if t.OS != runtime.GOOS || t.Arch != runtime.GOARCH {
		return nil // même commande de build que la cible native, vérifiée, elle
	}
	out, err := exec.Command(path).Output()
	if err != nil {
		return fmt.Errorf("%s : %w", path, err)
	}
	if !strings.HasPrefix(string(out), "svc "+version+" ") {
		return fmt.Errorf("%s : la version %s n'a pas été injectée (sortie : %q)", path, version, out)
	}
	return nil
}

func checksums(files []string) (string, error) {
	var lines []string
	for _, f := range files {
		data, err := os.ReadFile(f)
		if err != nil {
			return "", err
		}
		lines = append(lines, fmt.Sprintf("%x  %s\n", sha256.Sum256(data), filepath.Base(f)))
	}
	slices.SortFunc(lines, func(a, b string) int { return strings.Compare(a[66:], b[66:]) })
	return strings.Join(lines, ""), nil
}
```

```go
// Fichier : cmd/release/main_test.go
package main

import (
	"os"
	"os/exec"
	"path/filepath"
	"runtime"
	"strings"
	"testing"
)

// buildSvc construit le vrai paquet svc avec les options données, dans t.TempDir().
func buildSvc(t *testing.T, env []string, args ...string) string {
	t.Helper()
	out := filepath.Join(t.TempDir(), "svc")
	cmd := exec.Command("go", append(append([]string{"build"}, args...), "-o", out, "../svc")...)
	cmd.Env = append(os.Environ(), env...)
	if b, err := cmd.CombinedOutput(); err != nil {
		t.Fatalf("%v\n%s", err, b)
	}
	return out
}

func TestVerify(t *testing.T) {
	native := Target{runtime.GOOS, runtime.GOARCH}
	env := []string{"CGO_ENABLED=0", "GOOS=" + runtime.GOOS, "GOARCH=" + runtime.GOARCH}
	good := buildSvc(t, env, "-trimpath", "-ldflags", "-X main.version=v9.9.9")
	if err := Verify(good, "v9.9.9", native); err != nil {
		t.Fatalf("binaire correct refusé : %v", err)
	}
	for _, tc := range []struct {
		name, path, version, want string
		target                    Target
	}{
		{"mauvaise version", good, "v1.0.0", "pas été injectée", native},
		{"mauvaise cible", good, "v9.9.9", "GOARCH", Target{runtime.GOOS, "riscv64"}},
		{"sans -trimpath", buildSvc(t, env, "-ldflags", "-X main.version=v9.9.9"), "v9.9.9", "-trimpath", native},
		{"avec cgo", buildSvc(t, []string{"CGO_ENABLED=1"}, "-trimpath"), "dev", "CGO_ENABLED", native},
	} {
		t.Run(tc.name, func(t *testing.T) {
			err := Verify(tc.path, tc.version, tc.target)
			if err == nil || !strings.Contains(err.Error(), tc.want) {
				t.Fatalf("erreur %v, attendu %q", err, tc.want)
			}
		})
	}
}
```

Dans un dépôt Git propre, puis après une modification :

```
$ go run ./cmd/release v1.2.0
6fe0bb6349d9fce680f471117a229295969a43fe717c201e5bbb15351f32242f  svc-v1.2.0-darwin-arm64
1ff7998d7ab0d0b0a8226295e7065fbad2dc36bfec2abe2cf32ebb8be773a7b6  svc-v1.2.0-linux-amd64
704d39df39b59fbe23ecf2c035ab8d347bf2064a5f67f9440350b15651d32e0c  svc-v1.2.0-linux-arm64
d74ff2d3ff2e0048cef2beeddefbb58602e4e263df220849a7e4aa690d34b3bb  svc-v1.2.0-windows-amd64.exe
$ echo x >> cmd/svc/main.go && go run ./cmd/release v1.2.0
2026/11/03 13:36:07 arbre modifié, publication refusée :
 M cmd/svc/main.go
```

Les empreintes ne sont pas celles du cours : le build se fait cette fois dans un dépôt Git, et le commit est
inscrit dans le binaire (leçon 1).

**La vérification impossible.** `go version -m` montre les `-ldflags` d'un binaire construit **sans**
`-trimpath` :

```
	build	-ldflags="-s -w -X main.version=v1.2.0"
```

**Avec** `-trimpath`, la ligne disparaît : l'outil `go` n'enregistre pas les `-ldflags`, qui peuvent contenir
des chemins de la machine de build. On ne peut donc pas lire la version injectée dans les informations de
build.

Chercher la chaîne `v1.2.0` dans le fichier est une fausse bonne idée. La première version de la
vérification le faisait, et le test « mauvaise version » l'a pris en défaut : la chaîne `v1.0.0` figure dans
**tout** binaire Go récent, dans le chemin du paquet `crypto/internal/entropy/v1.0.0` de la bibliothèque
standard.

D'où la solution retenue. On **exécute** le binaire natif et on contrôle sa sortie. Pour les autres cibles, on
s'appuie sur leurs réglages vérifiés et sur le fait qu'elles sont construites par **la même** fonction
`build`.

Le reste du code :

- **`errgroup.SetLimit(4)`** : quatre compilations au plus à la fois. Chacune occupe déjà plusieurs cœurs.
- **`clean`** refuse de publier hors d'un dépôt Git, ou si l'arbre est modifié. Un binaire publié doit
  correspondre à un commit.
- **`TestVerify`** construit le **vrai** paquet `svc` dans `t.TempDir()`, de plusieurs façons. Il vérifie
  que chaque défaut est détecté et que le message d'erreur dit lequel. On ne teste pas `run` en entier :
  il faudrait un dépôt Git complet, et `clean` se teste à la main comme ci-dessus.

---

## Défi — Le même binaire, ailleurs

**1.** Localement, puis dans l'image officielle :

```
$ ./build.sh v1.2.0 | sed -n 5,7p
2a3380e58b732d8d	svc-v1.2.0-linux-amd64
1d3cef43c1ee2401	svc-v1.2.0-linux-arm64
42f428b4d143a8f7	svc-v1.2.0-windows-amd64.exe
$ docker run --rm -u $(id -u):$(id -g) -e HOME=/tmp -e GOCACHE=/tmp/cache \
    -v "$PWD":/src -w /src golang:1.27.3 ./build.sh v1.2.0 | sed -n 5,7p
2a3380e58b732d8d	svc-v1.2.0-linux-amd64
1d3cef43c1ee2401	svc-v1.2.0-linux-arm64
42f428b4d143a8f7	svc-v1.2.0-windows-amd64.exe
```

Les binaires sont **identiques**, octet pour octet, alors que la machine, le système, le répertoire (`/src`
contre un répertoire personnel) et l'utilisateur sont différents. `-u $(id -u)` évite que les fichiers de
`dist/` appartiennent à `root` sur la machine hôte.

**2.**

| Variante | Local | Conteneur |
|---|---|---|
| sans `-trimpath` | `fb0fa672af054d9a` | `c6263814b07d389e` |
| `CGO_ENABLED=1` pour `linux/amd64` | `98c1abf883c14821` | `328484817e740f1e` |

- **Sans `-trimpath`**, les chemins des sources diffèrent (`/src` contre le répertoire local), et les binaires
  aussi.
- **Avec cgo**, le compilateur C, la libc et l'éditeur de liens du système entrent dans le binaire. Ils ne
  sont pas les mêmes sur la machine et dans l'image.

Une première tentative avait activé cgo pour **toutes** les cibles, et elle échoue sur `linux/arm64` :

```
gcc_arm64.S:36: Error: no such instruction: `stp x19,x20,[sp,'
```

Le `gcc` installé ne produit que du code amd64 : avec cgo, la cross-compilation exige une chaîne C par cible.

**3.** Avec une image plus ancienne, `golang:1.26.1-alpine3.22` (après `apk add bash`, le script étant en
Bash) :

```
GOTOOLCHAIN=local
go: go.mod requires go >= 1.27 (running go 1.26.1; GOTOOLCHAIN=local)
```

Les images officielles fixent `GOTOOLCHAIN=local` : pas de téléchargement d'une autre version de Go, et le
build refuse. Avec `-e GOTOOLCHAIN=auto` :

```
e2b00884cbd689f7	svc-v1.2.0-linux-amd64
2e9a5fb13f7ab4c1	svc-v1.2.0-linux-arm64
7ab9ac2276141a5d	svc-v1.2.0-windows-amd64.exe
svc v1.2.0 (construit le 2026-03-01, go1.27.0, linux/amd64, debug=false)
```

Le build réussit, mais avec **go1.27.0** : la ligne `go 1.27` demande « au moins 1.27 », et c'est la version
1.27.0 qui est téléchargée. Les empreintes changent. On ajoute alors `toolchain go1.27.3` à `go.mod` :

```
go: downloading go1.27.3 (linux/amd64)
2a3380e58b732d8d	svc-v1.2.0-linux-amd64
1d3cef43c1ee2401	svc-v1.2.0-linux-arm64
42f428b4d143a8f7	svc-v1.2.0-windows-amd64.exe
```

Même depuis une image Go 1.26 Alpine, on retrouve **exactement** les binaires de référence. La ligne
`toolchain` fixe la version de Go qui construit, et donc le binaire. Elle ne change pas elle-même
l'empreinte : le binaire local reste `2a3380e58b73`.

**4. Une procédure de vérification**, par exemple :

> 1. Récupérer le dépôt et se placer sur l'étiquette : `git clone … && git checkout v1.2.0`.
> 2. Vérifier que l'étiquette est bien celle annoncée : `git show v1.2.0`, et sa signature si elle est
>    signée (leçon 8).
> 3. Lire la version de Go dans `go.mod` (ligne `toolchain`).
> 4. Construire dans l'image officielle de cette version :
>    `docker run --rm -v "$PWD":/src -w /src golang:1.27.3 ./build.sh v1.2.0`.
> 5. Télécharger les binaires publiés et leur `SHA256SUMS`.
> 6. Vérifier les binaires téléchargés : `sha256sum -c SHA256SUMS`.
> 7. Comparer avec le `dist/SHA256SUMS` reconstruit : les deux fichiers doivent être identiques.
> 8. En cas de différence, comparer `go version -m` des deux binaires pour trouver le réglage qui diffère.
> 9. Signaler toute différence inexpliquée : le binaire publié ne vient peut-être pas de ces sources.

---

## Quiz

1. Le binaire contient le *runtime*, la bibliothèque standard et les dépendances : rien à installer. Il dépend
   de la libc si cgo est actif et qu'un paquet l'utilise (`net`, `os/user`), ce qui est le cas par défaut avec
   un compilateur C installé.
2. `GOOS` et `GOARCH`, rien d'autre : la chaîne de compilation de Go sait produire toutes les cibles.
3. Il désactive cgo : les paquets utilisent leur implémentation en Go pur, et le binaire est statique.
4. Un suffixe dans le nom du fichier (`_linux.go`, `_arm64.go`), ou une ligne `//go:build expression`
   avant la clause `package`.
5. Les mettre dans des fichiers marqués `//go:build integration`, et les lancer avec
   `go test -tags integration ./...`. Ne pas oublier de les lancer en CI.
6. Des variables `string` initialisées par une constante (ou sans initialisation). Les constantes, les
   variables initialisées par un appel et les noms inexistants sont ignorés sans message.
7. La table des symboles et les informations DWARF. On garde les traces de pile complètes. On perd l'essentiel
   du débogage : pas de variables dans `dlv`.
8. Les mêmes sources, la même version de Go et les mêmes options donnent les mêmes octets. Cela permet à
   n'importe qui de vérifier qu'un binaire publié vient bien du code annoncé.
9. Il remplace les chemins absolus des sources par des chemins de module : le binaire ne dépend plus du
   répertoire de build, et ne révèle plus sa structure. Il fait aussi omettre `-ldflags` des informations de
   build.
10. Chaque build serait différent : plus aucune reproductibilité. Si l'on veut une date, c'est celle **du
    commit**, déjà inscrite par l'outil `go` (`vcs.time`).
