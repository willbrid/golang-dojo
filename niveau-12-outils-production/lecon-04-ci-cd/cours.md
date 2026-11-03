# Leçon 4 — CI/CD

> Les leçons précédentes ont produit une liste de vérifications : formatage, `go vet`, `staticcheck`, tests
> sous `-race`, `govulncheck`, build reproductible, image. Faites à la main, elles finissent par être oubliées
> un vendredi soir. L'**intégration continue** (CI) les exécute à chaque modification proposée. Le
> **déploiement continu** (CD) transforme une étiquette de version en binaires et en image publiés, sans
> manipulation. Cette leçon écrit les deux avec GitHub Actions. Elle les écrit surtout de façon à ce que la CI
> fasse **exactement** ce que fait un développeur sur son poste.

## Objectifs

- Choisir les contrôles d'une CI Go et leur ordre.
- Écrire un script de CI exécutable localement, et un workflow GitHub Actions qui l'appelle.
- Lancer des tests d'intégration avec une base fournie par la CI.
- Publier binaires et image multi-architecture à partir d'une étiquette.
- Sécuriser les workflows : permissions minimales, actions épinglées, données non fiables.

## Explication

### Ce que la CI vérifie

| Contrôle | Commande | Ce qu'il attrape |
|---|---|---|
| dépendances à jour | `go mod tidy`, puis aucune différence | un `go.mod` qui ne correspond pas au code |
| formatage | `gofmt -l .` vide | du code non formaté (N1) |
| analyse | `go vet ./...` **et** avec chaque étiquette de build | erreurs probables (N5 L8), y compris derrière `//go:build` (leçon 2) |
| analyse approfondie | `staticcheck ./...` | code mort, API dépréciées, bugs subtils (N5 L8) |
| tests | `go test -race -shuffle=on ./...` | régressions, **courses de données**, dépendances à l'ordre des tests |
| vulnérabilités | `govulncheck ./...` | une dépendance vulnérable **réellement appelée** (leçon 8) |
| build | `go build -buildvcs=true -trimpath …` | ce qui ne compile pas, et une version absente (leçon 1) |

L'ordre va du **plus rapide** au plus lent : un fichier mal formaté doit échouer en une seconde, pas après dix
minutes de tests.

`-race` n'est pas une option de luxe. Une course de données sur un simple compteur `int`, incrémenté par
8 goroutines, **passe** trois fois sur trois sans `-race`, et donne même le bon total. Le détecteur la signale
dès la première exécution (mesuré dans l'exemple). Un test qui passe ne prouve pas l'absence de course ;
`-race` le dit.

`-shuffle=on` exécute les tests dans un ordre aléatoire, affiché en cas d'échec pour qu'on puisse le rejouer.
Un test qui ne passe qu'après un autre, parce qu'il compte sur un état laissé par celui-ci, est ainsi démasqué.

Les outils sont lancés par `go run module@version` : leur version est **fixée** dans le script, la même sur
tous les postes et en CI. On évite ainsi le piège de N5 L8 : un outil compilé avec une ancienne version de Go
qui échoue sur un module récent.

### Le même script partout

Un workflow qui contient vingt lignes de commandes devient vite une boîte noire. Quand il échoue, il faut
pousser des commits « fix ci » pour le déboguer. On met donc les contrôles dans un **script du dépôt**
(`ci.sh`), que le développeur lance avant de pousser, et que le workflow appelle **tel quel**. La CI n'a
plus que trois rôles : fournir la machine, la bonne version de Go et les services (une base de données).

### GitHub Actions en bref

Un **workflow** est un fichier YAML dans `.github/workflows/`.

- **Déclencheurs** (`on:`) : une poussée sur `main`, une *pull request*, une étiquette.
- **Jobs** : chacun s'exécute sur une machine neuve (`runs-on: ubuntu-latest`), en parallèle des autres, sauf
  s'il en attend un (`needs:`).
- **Étapes** (`steps:`) : des commandes (`run:`), ou des **actions** (`uses:`), c'est-à-dire des étapes
  réutilisables publiées dans un dépôt.

Les actions dont on a besoin :

- `actions/checkout` récupère le dépôt ;
- `actions/setup-go` installe Go. Avec `go-version-file: go.mod`, il lit la ligne `toolchain` : la CI utilise
  exactement la version du `go.mod` (leçon 2). Il met aussi en cache les modules et le cache de compilation
  (`cache: true` par défaut) ;
- `actions/upload-artifact` conserve des fichiers produits par un job : rapports, binaires.

Des **services** (`services:`) lancent des conteneurs à côté du job, par exemple un PostgreSQL pour les tests
d'intégration. Le job attend qu'ils soient en bonne santé.

### La publication (CD)

Le workflow de publication se déclenche sur une étiquette `v*.*.*` (leçon 1) :

1. il relance les contrôles : on ne publie jamais une étiquette qui ne les passe pas ;
2. il construit les binaires avec `build.sh` (leçon 2) et crée une publication GitHub avec les binaires et
   `SHA256SUMS`. Une étiquette qui contient un `-` (`v1.3.0-rc.1`) est marquée comme préversion ;
3. il construit l'image pour amd64 et arm64 (leçon 3, défi) et la pousse dans le registre du dépôt
   (`ghcr.io`), étiquetée `1.3.0` et `1.3`.

Le déploiement **sur les serveurs** est l'objet de la leçon 9. Ici, la CD s'arrête à des artefacts publiés,
versionnés et vérifiables.

### Sécuriser les workflows

Un workflow s'exécute avec des droits sur le dépôt, et parfois avec des secrets. C'est une cible.

- **Permissions minimales.** `permissions: contents: read` au niveau du workflow. Seul le job qui en a besoin
  reçoit davantage : `contents: write` pour créer une publication, `packages: write` pour pousser une image.
- **Des actions épinglées par empreinte de commit.** Une étiquette comme `@v7` peut être **déplacée** par
  l'auteur de l'action, ou par quelqu'un qui a volé son compte (leçon 1). Une empreinte complète
  (`@3d3c42e5…`) désigne un code immuable. On garde la version en commentaire (`# v7.0.1`) pour la lisibilité
  et les mises à jour.
- **`persist-credentials: false`** pour `checkout` : le jeton n'est pas laissé dans la configuration Git du
  dépôt récupéré, où une étape ultérieure pourrait le lire.
- **Les données non fiables ne vont jamais directement dans un script.** Le titre d'une *pull request*, le nom
  d'une branche ou le message d'un commit sont écrits par n'importe qui. `run: echo "${{
  github.event.pull_request.title }}"` est **remplacé avant** l'exécution du shell : un titre contenant
  `"; curl … | sh; "` est exécuté. On passe la valeur par une variable d'environnement (`env: TITLE: …`, puis
  `"$TITLE"` dans le script), qui n'est jamais interprétée comme du code.
- **`actionlint`** vérifie la syntaxe des workflows, les expressions, et signale justement ces injections.

### Rapide et fiable

- **Une CI lente est contournée.** On vise moins de 10 minutes : les caches de `setup-go`, des jobs
  parallèles, et les tests lents séparés derrière une étiquette `integration`.
- **`concurrency` avec `cancel-in-progress`** annule la vérification d'un commit déjà remplacé par un plus
  récent sur la même branche.
- **Un test instable** (*flaky*), qui échoue une fois sur vingt, est un bug à corriger immédiatement. Le
  relancer jusqu'à ce qu'il passe apprend à l'équipe à ignorer la CI.
- **`timeout-minutes`** : un test bloqué (un interblocage, N6 L10) ne doit pas occuper une machine pendant six
  heures, la durée maximale par défaut.

## Exemple

Le service de la leçon 2, avec un paquet testé, un test d'intégration, le script de CI et deux workflows.

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27

toolchain go1.27.3
```

```go
// Fichier : internal/stats/stats.go
// Package stats compte des requêtes par route, depuis plusieurs goroutines.
package stats

import "sync"

type Counter struct {
	mu sync.Mutex
	n  map[string]int
}

func New() *Counter { return &Counter{n: map[string]int{}} }

func (c *Counter) Inc(route string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n[route]++
}

func (c *Counter) Get(route string) int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.n[route]
}
```

```go
// Fichier : internal/stats/stats_test.go
package stats

import (
	"sync"
	"testing"
)

func TestConcurrentInc(t *testing.T) {
	c := New()
	var wg sync.WaitGroup
	for range 8 {
		wg.Go(func() {
			for range 1000 {
				c.Inc("/a")
			}
		})
	}
	wg.Wait()
	if got := c.Get("/a"); got != 8000 {
		t.Fatalf("Get = %d, attendu 8000", got)
	}
}
```

```go
// Fichier : internal/stats/db_integration_test.go
//go:build integration

package stats

import (
	"net"
	"net/url"
	"os"
	"testing"
	"time"
)

// Un test d'intégration minimal : la base fournie par la CI répond-elle ?
// (Un vrai service y lancerait sa suite de contrat du dépôt, N8 L7.)
func TestDatabaseReachable(t *testing.T) {
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Fatal("DATABASE_URL manquante : ce test exige une base")
	}
	u, err := url.Parse(dsn)
	if err != nil {
		t.Fatal(err)
	}
	conn, err := net.DialTimeout("tcp", u.Host, 3*time.Second)
	if err != nil {
		t.Fatalf("base injoignable : %v", err)
	}
	conn.Close()
}
```

```go
// Fichier : cmd/svc/main.go
// Commande svc : le service que la CI vérifie, construit et publie.
package main

import (
	"fmt"
	"net/http"

	"example.com/svc/internal/stats"
)

var version = "dev"

func main() {
	c := stats.New()
	http.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
		c.Inc(r.URL.Path)
		fmt.Fprintf(w, "svc %s : %d visites\n", version, c.Get(r.URL.Path))
	})
	http.ListenAndServe(":8080", nil)
}
```

```bash
#!/usr/bin/env bash
# ci.sh : les contrôles de la CI, lancés à l'identique sur un poste de développement.
set -euo pipefail

step() { printf '== %-28s' "$1"; }
ok() { echo "ok"; }

step "go mod tidy (inchangé)"
go mod tidy
git diff --quiet -- go.mod go.sum 2>/dev/null || { echo "go.mod/go.sum pas à jour : lancer go mod tidy"; exit 1; }
ok

step "gofmt"
unformatted=$(gofmt -l .)
[ -z "$unformatted" ] || { echo "fichiers à formater : $unformatted"; exit 1; }
ok

step "go vet (avec et sans tags)"
go vet ./...
go vet -tags integration ./... # sinon le code derrière l'étiquette n'est jamais vérifié (leçon 2)
ok

step "staticcheck"
go run honnef.co/go/tools/cmd/staticcheck@v0.8.2 ./...
ok

step "go test -race"
go test -race -shuffle=on -count=1 -coverprofile=cover.out ./... > test.log || { echo; cat test.log; exit 1; }
ok

step "govulncheck"
go run golang.org/x/vuln/cmd/govulncheck@v1.6.0 ./... > vuln.log || { echo; cat vuln.log; exit 1; }
ok

step "go build"
CGO_ENABLED=0 go build -buildvcs=true -trimpath -ldflags "-s -w" -o bin/svc ./cmd/svc
ok
```

`build.sh` est celui de la leçon 2, la date étant désormais celle du commit (`git log -1 --format=%cs`). Le `Dockerfile` est celui du défi de la leçon 3 (`--platform=$BUILDPLATFORM`).

```yaml
# .github/workflows/ci.yml
name: ci

on:
  push:
    branches: [main]
  pull_request:

# Une nouvelle poussée sur la même branche annule la vérification précédente, devenue inutile.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Le jeton de la CI ne peut que lire le dépôt.
permissions:
  contents: read

jobs:
  checks:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-go@b7ad1dad31e06c5925ef5d2fc7ad053ef454303e # v7.0.0
        with:
          go-version-file: go.mod # lit la ligne toolchain
      - run: ./ci.sh
      - uses: actions/upload-artifact@cf430e030ddbb5b0abf93d22962f4752f3646cd9 # v7.0.2
        if: always()
        with:
          name: rapports
          path: |
            cover.out
            test.log
            vuln.log

  integration:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: ci
          POSTGRES_PASSWORD: ci # une base jetable, créée pour ce job : pas un secret
          POSTGRES_DB: ci
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U ci"
          --health-interval 2s
          --health-timeout 3s
          --health-retries 15
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-go@b7ad1dad31e06c5925ef5d2fc7ad053ef454303e # v7.0.0
        with:
          go-version-file: go.mod
      - run: go test -race -tags integration -count=1 ./...
        env:
          DATABASE_URL: postgres://ci:ci@localhost:5432/ci?sslmode=disable

  build:
    needs: [checks]
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-go@b7ad1dad31e06c5925ef5d2fc7ad053ef454303e # v7.0.0
        with:
          go-version-file: go.mod
      - run: ./build.sh "dev-${GITHUB_SHA::12}"
      - uses: actions/upload-artifact@cf430e030ddbb5b0abf93d22962f4752f3646cd9 # v7.0.2
        with:
          name: binaires
          path: dist/
          retention-days: 7
```

```yaml
# .github/workflows/release.yml
name: release

on:
  push:
    tags: ["v*.*.*"]

permissions:
  contents: read

jobs:
  binaries:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: write # créer la publication GitHub et y joindre les fichiers
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/setup-go@b7ad1dad31e06c5925ef5d2fc7ad053ef454303e # v7.0.0
        with:
          go-version-file: go.mod
      - run: ./ci.sh # on ne publie jamais une étiquette qui ne passe pas les contrôles
      - run: ./build.sh "$GITHUB_REF_NAME"
      - uses: softprops/action-gh-release@efb35369e0ad2afab669f228072c1b0d510eae64 # v3.0.3
        with:
          files: dist/*
          generate_release_notes: true
          prerelease: ${{ contains(github.ref_name, '-') }}

  image:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: read
      packages: write # pousser l'image dans le registre du dépôt (ghcr.io)
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: docker/setup-buildx-action@f87e5991a6d7451dcb8d9637bfbc97413f497069 # v4.4.1
      - uses: docker/login-action@81c6f93d4bc991e510b342a00b419b3bb4efc5c6 # v4.7.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@dc802804100637a589fabce1cb79ff13a1411302 # v6.2.0
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
      - uses: docker/build-push-action@c3c9e263c25d99ce0380d002d59b67737d91b0dc # v7.4.0
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: VERSION=${{ github.ref_name }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Le script, sur le poste de développement :

```
$ ./ci.sh
== go mod tidy (inchangé)     ok
== gofmt                       ok
== go vet (avec et sans tags)  ok
== staticcheck                 ok
== go test -race               ok
== govulncheck                 ok
== go build                    ok
$ cat test.log
	example.com/svc/cmd/svc		coverage: 0.0% of statements
ok  	example.com/svc/internal/stats	1.031s	coverage: 100.0% of statements
$ cat vuln.log
No vulnerabilities found.
```

Le test d'intégration, contre une base locale (le conteneur PostgreSQL du niveau 8) :

```
$ DATABASE_URL=postgres://dojo:dojo@127.0.0.1:55439/dojo go test -race -tags integration -count=1 -v ./internal/stats/
=== RUN   TestDatabaseReachable
--- PASS: TestDatabaseReachable (0.00s)
=== RUN   TestConcurrentInc
--- PASS: TestConcurrentInc (0.01s)
ok  	example.com/svc/internal/stats	1.021s
```

Les contrôles qui échouent. Chaque défaut est introduit puis retiré :

```
# le verrou oublié dans Inc : la map est écrite par 8 goroutines
== go test -race
WARNING: DATA RACE
# (sans -race, le runtime détecte ici l'écriture concurrente dans une map : fatal error: concurrent map writes)

# un fichier mal formaté
== gofmt                       fichiers à formater : internal/stats/mal.go

# un require inutile validé dans go.mod
== go mod tidy (inchangé)     go.mod/go.sum pas à jour : lancer go mod tidy
```

Et la course sur un `int`, que seul `-race` voit :

```go
total := 0
for range 8 {
	wg.Go(func() {
		for range 1000 {
			total++ // course sur un int
		}
	})
}
```

```
$ go test -count=1 -v .        # trois fois
    rc_test.go:19: total = 8000 ok
    rc_test.go:19: total = 8000 ok
    rc_test.go:19: total = 8000 ok
$ go test -race -count=1 .
WARNING: DATA RACE
```

Les workflows, vérifiés par `actionlint` (v1.7.7) :

```
$ actionlint && echo "actionlint : aucun problème"
actionlint : aucun problème
```

Sur un workflow de test qui insère le titre d'une *pull request* dans un script :

```
.github/workflows/bad.yml:10:30: "github.event.pull_request.title" is potentially untrusted. avoid using it directly in inline scripts. instead, pass it through an environment variable. […] [expression]
```

> **Ce qui a été vérifié, et ce qui ne l'a pas été.** Le script `ci.sh`, `build.sh`, le Dockerfile et le test
> d'intégration ont été exécutés. Les workflows ont été validés par `actionlint`, et chaque entrée utilisée
> (`go-version-file`, `files`, `platforms`…) a été vérifiée dans le fichier `action.yml` de l'action, à
> l'empreinte épinglée. Ils n'ont **pas** été exécutés sur GitHub depuis ce cours : la première exécution
> dans ton dépôt est le vrai test.
>
> Point d'attention : `actionlint` ne connaît pas les entrées des versions les plus récentes de toutes les
> actions. Une faute de frappe (`go-version-fil`) **n'a pas** été signalée lors de l'essai. Elle serait
> ignorée par GitHub avec un simple avertissement, et `setup-go` installerait une autre version de Go.

## Explication du code

| Élément | Rôle |
|---|---|
| `set -euo pipefail` | Le script s'arrête à la première commande qui échoue, à la première variable non définie, et si une commande d'un tube échoue. Sans cela, un contrôle en échec pourrait être suivi d'un « ok ». |
| `go mod tidy` puis `git diff --quiet` | Compare au **commit**. La CI part d'un dépôt propre : toute différence signifie un `go.mod` validé qui n'est pas à jour. |
| `go run …/staticcheck@v0.8.2` | La version de l'outil est fixée par le script, et compilée avec la version de Go du projet. |
| `-coverprofile=cover.out` | Le rapport de couverture (N5 L5), conservé comme artefact par la CI. |
| `//go:build integration` | Le test n'est compilé qu'avec `-tags integration` : `go test ./...` ne demande pas de base. `ci.sh` le vérifie tout de même avec `go vet -tags integration`. |
| `t.Fatal` si `DATABASE_URL` manque | Avec l'étiquette, l'absence de base est une **erreur**, pas un `t.Skip`. Un test d'intégration qui se saute en silence en CI ne teste rien. |
| `concurrency`, `cancel-in-progress` | Une vérification par branche à la fois ; la plus ancienne est annulée. |
| `permissions` au niveau du workflow et du job | Lecture seule par défaut ; écriture seulement pour le job qui publie. |
| `@3d3c42e5… # v7.0.1` | Une action épinglée par empreinte de commit, avec la version en commentaire. |
| `services: postgres:` et `options: --health-cmd …` | Une base jetable, prête avant la première étape. Elle est joignable sur `localhost:5432` parce que le port est publié. |
| `needs: [checks]` | Le job `build` ne démarre que si les contrôles ont réussi. |
| `"dev-${GITHUB_SHA::12}"` | Les variables d'environnement de GitHub (`GITHUB_SHA`, `GITHUB_REF_NAME`) s'utilisent dans le shell, sans `${{ }}`. |
| `prerelease: ${{ contains(github.ref_name, '-') }}` | `v1.3.0-rc.1` devient une préversion. Ici `${{ }}` est sans danger : la valeur n'est pas insérée dans un script. |
| `type=semver,pattern={{version}}` | L'étiquette `v1.3.0` donne les étiquettes d'image `1.3.0` et `1.3`. |
| `cache-from/cache-to: type=gha` | Le cache des couches Docker (leçon 3) conservé entre deux exécutions de la CI. |

## Erreurs fréquentes

1. **Une CI qui ne lance pas `-race`**, ou pas `go vet` avec les étiquettes de build utilisées.
2. **Des commandes différentes en CI et en local** : « ça passe chez moi ».
3. **Des outils installés en `@latest`** : la CI casse un matin sans aucun changement dans le dépôt.
4. **Des actions référencées par une étiquette déplaçable** (`@v7`, `@main`).
5. **`permissions` absent** : selon les réglages du dépôt, le jeton peut avoir des droits d'écriture partout.
6. **`${{ github.event… }}` dans un `run:`** : injection de commandes par un titre ou un nom de branche.
7. **Des tests instables relancés** jusqu'à ce qu'ils passent.
8. **Publier depuis un poste de développement** : un binaire `+dirty`, sans traçabilité.
9. **Un test d'intégration qui se saute** quand la base manque, et passe pour un succès.

## Bonnes pratiques Go

- Un `ci.sh` (ou un `Makefile`) dans le dépôt, appelé tel quel par la CI.
- `go-version-file: go.mod`, avec une ligne `toolchain` : une seule source pour la version de Go.
- Contrôles du plus rapide au plus lent : `tidy`, `gofmt`, `vet`, `staticcheck`, tests `-race -shuffle=on`,
  `govulncheck`, build.
- Tests d'intégration derrière une étiquette, avec des services fournis par la CI.
- Publication sur étiquette, après les mêmes contrôles ; binaires, `SHA256SUMS` et image multi-architecture.
- Permissions minimales, actions épinglées par empreinte, `actionlint`, aucune donnée non fiable dans un
  script.
- `timeout-minutes` sur chaque job, `concurrency` sur chaque workflow.

## Ce que je dois retenir

- La CI exécute à chaque modification ce qu'on oublierait à la main ; la CD transforme une étiquette en
  artefacts publiés.
- Le même script en local et en CI ; la CI fournit la machine, la version de Go et les services.
- `-race` trouve des courses qu'aucun test ne révèle (8000 = 8000, trois fois, et pourtant une course).
- Un workflow est du code exécuté avec des droits : permissions minimales, actions épinglées, pas
  d'injection.
- Tout n'a pas été exécuté ici : la première exécution d'un workflow dans un vrai dépôt reste à faire.

---

➡️ [Exercices](exercices.md)
