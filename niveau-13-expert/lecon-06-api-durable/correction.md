# Module 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le module du cours (`example.com/api`) reçoit les paquets des exercices, et l'exercice intermédiaire y
ajoute un second module, `example.com/api/v2`, dans le sous-répertoire `v2/` :

```go.mod
// Fichier : go.mod
module example.com/api

go 1.27

require example.com/api/v2 v2.0.0

replace example.com/api/v2 => ./v2
```

## E1 — Prédire, puis vérifier

```go
// Fichier : e1/geom/geom.go
// Package geom : un autre paquet, cible de l'alias du changement 5.
package geom

type Point struct{ X, Y float64 }
```

```go
// Fichier : e1/old/lib/lib.go
// Package lib : la version publiée.
package lib

type Config struct{ Name string }

func Load() *Config { return &Config{Name: "défaut"} } // 1

type Server struct{ addr string } // 2

const MaxItems = 100 // 3

type Key struct { // 4 : comparable
	Name string
}

type Point struct{ X, Y float64 } // 5

func Repeat(s string, n int) string { // 6
	out := ""
	for range n {
		out += s
	}
	return out
}
```

```go
// Fichier : e1/new/lib/lib.go
// Package lib : la version modifiée.
package lib

import "example.com/api/e1/geom"

type Config struct{ Name string }

func Load() Config { return Config{Name: "défaut"} } // 1 : *Config → Config

type Server struct{ addr string }

func (s *Server) Addr() string { return s.addr } // 2 : méthode ajoutée à une struct

const MaxItems = 200 // 3 : valeur changée

type Key struct { // 4 : un champ non exporté de type slice : Key n'est plus comparable
	Name string
	tags []string
}

type Point = geom.Point // 5 : alias vers un autre paquet

func Repeat(s string, n int64) string { // 6 : int → int64
	out := ""
	for range n {
		out += s
	}
	return out
}
```

```go
// Fichier : e1/client/client.go
// Package client : un utilisateur de lib, écrit pour l'ancienne version.
package client

import "example.com/api/e1/old/lib"

func Use() {
	c := lib.Load()
	c.Name = "x"              // 1 : modifie la configuration partagée par le pointeur
	var buf [lib.MaxItems]int // 3 : une taille de tableau qui dépend de la constante
	_ = buf
	seen := map[lib.Key]bool{{Name: "a"}: true} // 4 : Key comme clé de map, ou ==
	_ = seen[lib.Key{Name: "a"}] && lib.Key{Name: "a"} == lib.Key{Name: "b"}
	var p lib.Point = lib.Point{X: 1, Y: 2} // 5
	_ = p
	n := 3
	_ = lib.Repeat("ab", n) // 6 : une variable int
}
```

```
$ apidiff example.com/api/e1/old/lib example.com/api/e1/new/lib
Incompatible changes:
- Key: old is comparable, new is not
- Load: changed from func() *Config to func() Config
- MaxItems: value changed from 100 to 200
- Point: changed from Point to example.com/api/e1/geom.Point
- Repeat: changed from func(string, int) string to func(string, int64) string
Compatible changes:
- (*Server).Addr: added
$ go build ./client2/        # le même client, contre e1/new/lib
client2/client.go:11:14: invalid map key type lib.Key
client2/client.go:12:34: invalid operation: lib.Key{…} == lib.Key{…} (struct containing []string cannot be compared)
client2/client.go:16:23: cannot use n (variable of type int) as int64 value in argument to lib.Repeat
```

| # | Changement | `apidiff` | Le client compile ? | Ce qui casse vraiment |
|---|---|---|---|---|
| 1 | `*Config` → `Config` | incompatible | **oui** | le comportement : `c.Name = "x"` modifiait la configuration partagée, il ne modifie plus qu'une copie |
| 2 | méthode ajoutée à une struct | compatible | oui | rien, sauf une struct incluse dans une autre qui définissait déjà une méthode de ce nom |
| 3 | `MaxItems` 100 → 200 | incompatible | **oui** | la taille d'un tableau `[lib.MaxItems]int`, compilée chez le client, et tout comportement qui en dépend |
| 4 | champ non exporté `[]string` ajouté | incompatible : « old is comparable, new is not » | non | `==` et l'usage comme clé de map : un champ **invisible** a cassé le client |
| 5 | alias vers un autre paquet | incompatible | **oui** | les clients qui implémentent des méthodes sur ce type, ou en dépendent par son chemin ; pas ce client-ci |
| 6 | `int` → `int64` | incompatible | non | tout appelant qui passe une variable `int` |

**`apidiff` a vu les six.** C'est le compilateur qui en a manqué trois (1, 3, 5) : le client compile, mais le
programme ne fait plus la même chose. D'où la règle : **compiler les clients ne suffit pas** à prouver la
compatibilité. Le changement 4 est le plus traître pour l'auteur : il ajoute un champ **privé**, ce qui semble
ne regarder personne, et il casse tous ceux qui comparaient des valeurs.

## E2 — Prévoir les paramètres futurs

```go
// Fichier : e1/optstruct/client.go
// Package optstruct : des paramètres futurs dans une struct d'options (E2).
package optstruct

import (
	"log/slog"
	"time"
)

// Options : on y ajoute des champs en v1.2 sans rien casser, à condition que leur valeur ZÉRO garde le
// comportement d'avant (d'où les « 0 = défaut » ci-dessous).
type Options struct {
	Timeout time.Duration // 0 : 10 s
	Retries int           // 0 : aucun réessai ; -1 n'a pas de sens et est refusé
	Logger  *slog.Logger  // nil : aucun journal
}

type Client struct {
	addr string
	opts Options
}

func NewClient(addr string, o Options) *Client {
	if o.Timeout == 0 {
		o.Timeout = 10 * time.Second
	}
	if o.Logger == nil {
		o.Logger = slog.New(slog.DiscardHandler)
	}
	return &Client{addr: addr, opts: o}
}

func (c *Client) Timeout() time.Duration { return c.opts.Timeout }
```

```go
// Fichier : e1/optfunc/client.go
// Package optfunc : des paramètres futurs par options fonctionnelles (E2).
package optfunc

import (
	"log/slog"
	"time"
)

type Client struct {
	addr    string
	timeout time.Duration
	retries int
	log     *slog.Logger
}

// Option modifie la configuration ; en ajouter une en v1.2 ne change aucune signature existante.
type Option func(*Client)

func WithTimeout(d time.Duration) Option { return func(c *Client) { c.timeout = d } }
func WithRetries(n int) Option           { return func(c *Client) { c.retries = n } }
func WithLogger(l *slog.Logger) Option   { return func(c *Client) { c.log = l } }

// NewClient : les valeurs par défaut sont posées AVANT les options ; une option peut donc fixer
// explicitement une valeur nulle (WithTimeout(0) : aucun délai), ce que la struct ne permet pas.
func NewClient(addr string, opts ...Option) *Client {
	c := &Client{addr: addr, timeout: 10 * time.Second, log: slog.New(slog.DiscardHandler)}
	for _, o := range opts {
		o(c)
	}
	return c
}

func (c *Client) Timeout() time.Duration { return c.timeout }
```

```go
// Fichier : e1/optfunc/client_test.go
package optfunc

import (
	"testing"
	"time"

	"example.com/api/e1/optstruct"
)

func TestDefaultsAndZero(t *testing.T) {
	if NewClient("a").Timeout() != 10*time.Second || NewClient("a", WithTimeout(0)).Timeout() != 0 {
		t.Fatal("options fonctionnelles")
	}
	// Avec la struct, « zéro » veut dire « défaut » : impossible de demander un délai nul.
	if optstruct.NewClient("a", optstruct.Options{Timeout: 0}).Timeout() != 10*time.Second {
		t.Fatal("struct")
	}
}
```

**Ajouter une option en v1.2** : un champ dans `Options`, ou une fonction `WithX` de plus. Dans les deux cas,
aucune signature ne change.

**La valeur zéro** distingue les deux conceptions. Avec la struct, `Timeout: 0` veut dire « par défaut » (le
test le vérifie). On ne peut donc **pas** demander un délai nul, sauf avec un pointeur ou un champ en plus.
Avec les options fonctionnelles, les valeurs par défaut sont posées **avant** les options, et
`WithTimeout(0)` demande explicitement 0.

**Le choix** :
- **La struct** est plus simple, se lit d'un coup d'œil, et s'affiche bien dans la documentation. C'est le
  choix par défaut quand la valeur zéro de chaque champ peut signifier « par défaut ».
- **Les options fonctionnelles** valent leur poids quand une valeur nulle est significative, ou quand une
  option a une logique (valider, combiner) : par exemple `grpc.Dial(addr, opts...)`.

## E3 — Migrer une constante et un type

```go
// Fichier : e1/cfg/cfg.go
// Package cfg : une constante et un type renommés, avec leurs anciens noms gardés pour la transition.
package cfg

import "time"

const DefaultTimeout = 5 * time.Second

// Deprecated: utiliser DefaultTimeout.
//
//go:fix inline
const Timeout = DefaultTimeout

type Options struct{ Retries int }

// Deprecated: utiliser Options.
//
//go:fix inline
type Settings = Options
```

```go
// Fichier : e1/use/use.go
// Package use : un utilisateur des anciens noms.
package use

import (
	"time"

	"example.com/api/e1/cfg"
)

func Defaults() (time.Duration, cfg.Settings) {
	return cfg.Timeout, cfg.Settings{Retries: 3}
}
```

```
$ go fix -diff ./e1/use/
-func Defaults() (time.Duration, cfg.Settings) {
-	return cfg.Timeout, cfg.Settings{Retries: 3}
+func Defaults() (time.Duration, cfg.Options) {
+	return cfg.DefaultTimeout, cfg.Options{Retries: 3}
 }
```

`//go:fix inline` fonctionne sur une **constante** définie par une autre constante, et sur un **alias** de
type. `go fix` remplace l'ancien nom par le nouveau partout, signatures comprises. C'est la façon de renommer
un identifiant d'une API publiée : garder l'ancien nom comme alias déprécié pendant la transition, et laisser
`go fix` faire la migration.

## E4 — `apidiff` en CI

```bash
#!/usr/bin/env bash
# api-check.sh : compare l'API du module courant à celle de la dernière étiquette ; échoue si un
# changement incompatible apparaît sans changement de chemin de module (/vN).
set -euo pipefail
module=$(go list -m)
tag=$(git describe --tags --abbrev=0 2>/dev/null) || { echo "aucune étiquette : rien à comparer"; exit 0; }
tmp=$(mktemp -d)
trap 'git worktree remove --force "$tmp/old" >/dev/null 2>&1; rm -rf "$tmp"' EXIT
git worktree add --quiet --detach "$tmp/old" "$tag"
oldmod=$(cd "$tmp/old" && go list -m)
if [ "$oldmod" != "$module" ]; then
	echo "chemin de module changé ($oldmod → $module) : nouvelle version majeure, pas de comparaison"
	exit 0
fi
(cd "$tmp/old" && apidiff -m -w "$tmp/old.api" "$module")
report=$(apidiff -m "$tmp/old.api" "$module")
echo "API comparée à $tag :"
echo "${report:-(aucun changement)}"
if grep -q "^Incompatible changes:" <<<"$report"; then
	echo "ÉCHEC : changement incompatible dans une version majeure inchangée"
	exit 1
fi
```

Sur un dépôt de test, avec une étiquette `v1.0.0` :

```
== 1. ajout compatible
API comparée à v1.0.0 :
Compatible changes:
- ./lib.Sub: added
rc=0
== 2. signature changée
API comparée à v1.0.0 :
Incompatible changes:
- ./lib.Add: changed from func(int, int) int to func(int64, int64) int64
ÉCHEC : changement incompatible dans une version majeure inchangée
rc=1
== 3. même changement, module /v2
chemin de module changé (example.com/checked → example.com/checked/v2) : nouvelle version majeure, pas de comparaison
rc=0
```

- **`git worktree add`** extrait l'étiquette dans un répertoire temporaire, sans toucher à l'arbre de travail.
  Le `trap` le supprime, même en cas d'échec : `git worktree list` n'en montre plus aucun après les essais.
- **`apidiff -m -w`** écrit l'API de **tout le module** à l'étiquette, dans un fichier. `apidiff -m` la compare
  ensuite au module courant. Les deux versions ont le même chemin de module : on ne peut pas les charger
  ensemble, d'où le passage par un fichier.
- **Un chemin de module changé** (`/v2`) est le signe d'une version majeure assumée : le script laisse passer.

## E5 — Renommer un champ JSON sans casser

```go
// Fichier : e1/rename/doc.go
// Package rename : la transition d'un champ JSON (E5).
package rename
```

```go
// Fichier : e1/rename/rename_test.go
package rename

import (
	"encoding/json"
	"fmt"
	"testing"
)

// E5 : renommer total en amount_cents en trois versions du serveur.

type oldClient struct { // ce que connaît l'ancien client
	Total int `json:"total"`
}

type newClient struct { // le nouveau client lit le nouveau champ, et l'ancien s'il manque
	AmountCents *int `json:"amount_cents"`
	Total       *int `json:"total"`
}

func (c newClient) amount() (int, bool) {
	switch {
	case c.AmountCents != nil:
		return *c.AmountCents, true
	case c.Total != nil:
		return *c.Total, true
	}
	return 0, false
}

var servers = []struct{ name, body string }{
	{"v1 : total seul", `{"total":1250}`},
	{"v2 : les deux", `{"total":1250,"amount_cents":1250}`}, // transition : ancien et nouveau clients servis
	{"v3 : amount_cents seul", `{"amount_cents":1250}`},     // seulement quand plus aucun ancien client n'existe
}

func TestTransition(t *testing.T) {
	for _, s := range servers {
		var o oldClient
		json.Unmarshal([]byte(s.body), &o)
		var n newClient
		json.Unmarshal([]byte(s.body), &n)
		amt, ok := n.amount()
		fmt.Printf("%-24s ancien client : total=%-4d  nouveau client : %d (trouvé %v)\n", s.name, o.Total, amt, ok)
		if !ok || amt != 1250 {
			t.Errorf("%s : le nouveau client ne trouve pas le montant", s.name)
		}
	}
}
```

```
v1 : total seul          ancien client : total=1250  nouveau client : 1250 (trouvé true)
v2 : les deux            ancien client : total=1250  nouveau client : 1250 (trouvé true)
v3 : amount_cents seul   ancien client : total=0     nouveau client : 1250 (trouvé true)
```

- **Le nouveau client** lit `amount_cents`, et retombe sur `total` s'il manque. Il fonctionne avec les trois
  versions du serveur. Les **pointeurs** distinguent « absent » de « zéro ».
- **L'ancien client** fonctionne avec v1 et v2. Avec v3, il lit `total=0` **sans erreur** : c'est le défaut du
  cours, dont la transition ne protège qu'à une condition.
- **La condition** : on ne déploie la v3 que quand **plus aucun ancien client** n'est en service. On le sait par
  les métriques (la version des clients, en en-tête), par les contrats du défi, ou par une date de fin de
  service annoncée. Pour une appli mobile, cela peut prendre des mois.

---

## Exercice intermédiaire — Une version 2 de `geo`

```go.mod
// Fichier : v2/go.mod
module example.com/api/v2

go 1.27
```

```go
// Fichier : v2/geo/geo.go
// Package geo : la version 2 (module example.com/api/v2).
package geo

import (
	"context"
	"errors"
	"math"
)

type Point struct{ Lat, Lon, Alt float64 }

// Store prend désormais un contexte : c'est le changement qui justifie la v2.
type Store interface {
	Get(ctx context.Context, name string) (Point, error)
	Put(ctx context.Context, name string, p Point) error
}

var ErrNotFound = errors.New("geo : point inconnu")

type Unit int

const (
	Kilometers Unit = iota
	Miles
)

// Calculator calcule les distances dans une unité choisie une fois.
type Calculator struct{ unit Unit }

func NewCalculator(u Unit) Calculator { return Calculator{unit: u} }

func (c Calculator) Distance(a, b Point) float64 {
	const r = 6371.0
	rad := math.Pi / 180
	dLat, dLon := (b.Lat-a.Lat)*rad, (b.Lon-a.Lon)*rad
	h := math.Sin(dLat/2)*math.Sin(dLat/2) + math.Cos(a.Lat*rad)*math.Cos(b.Lat*rad)*math.Sin(dLon/2)*math.Sin(dLon/2)
	d := 2 * r * math.Asin(math.Sqrt(h))
	if c.unit == Miles {
		return d / 1.609344
	}
	return d
}

// Nearest (l'ancienne NearestContext, renommée).
func Nearest(ctx context.Context, s Store, c Calculator, p Point, names []string) (string, error) {
	best, bestD := "", -1.0
	for _, n := range names {
		if err := ctx.Err(); err != nil {
			return "", err
		}
		q, err := s.Get(ctx, n)
		if err != nil {
			return "", err
		}
		if d := c.Distance(p, q); bestD < 0 || d < bestD {
			best, bestD = n, d
		}
	}
	return best, nil
}
```

La v1 (`release/geo`, devenue v1.2.0) est réécrite **par-dessus** la v2 :

```go
// Fichier : release/geo/geo.go
// Package geo : la version v1.1.0, compatible avec v1.0.0.
package geo

import (
	"context"
	"errors"

	geo2 "example.com/api/v2/geo"
)

// Point est un point en coordonnées géographiques (degrés).
//
// Depuis v1.1.0, Alt donne l'altitude en mètres. Ajouter un champ est permis par la promesse de
// compatibilité de Go : les littéraux sans noms de champs (geo.Point{1, 2}) ne sont pas protégés,
// et go vet les signale.
type Point struct {
	Lat, Lon float64
	Alt      float64
}

// Store mémorise des points nommés. L'interface ne change JAMAIS dans une version mineure :
// les implémentations des utilisateurs cesseraient de compiler.
type Store interface {
	Get(name string) (Point, error)
	Put(name string, p Point) error
}

// Deleter est une capacité optionnelle d'un Store (v1.1.0).
type Deleter interface {
	Delete(name string) error
}

var (
	ErrNotFound     = errors.New("geo : point inconnu")
	ErrNotSupported = errors.New("geo : opération non prise en charge par ce Store")
)

// Delete supprime name de s si s sait le faire (v1.1.0).
func Delete(s Store, name string) error {
	if d, ok := s.(Deleter); ok {
		return d.Delete(name)
	}
	return ErrNotSupported
}

// Unit est une unité de distance (v1.1.0).
type Unit int

const (
	Kilometers Unit = iota
	Miles
)

// Distance renvoie la distance en kilomètres entre deux points. Sa signature ne change pas.
func Distance(a, b Point) float64 { return haversine(a, b) }

// DistanceIn renvoie la distance dans l'unité u (v1.1.0).
func DistanceIn(a, b Point, u Unit) float64 {
	if u == Miles {
		return haversine(a, b) / 1.609344
	}
	return haversine(a, b)
}

// Nearest renvoie le nom du point de s le plus proche de p parmi names.
//
// Deprecated: utiliser NearestContext, qui accepte l'annulation. Nearest sera retirée en v2.
//
//go:fix inline
func Nearest(s Store, p Point, names []string) (string, error) {
	return NearestContext(context.Background(), s, p, names)
}

// NearestContext est Nearest avec un contexte (v1.1.0). Depuis v1.2.0, elle est implémentée par la v2.
//
// Deprecated: utiliser example.com/api/v2/geo.Nearest.
func NearestContext(ctx context.Context, s Store, p Point, names []string) (string, error) {
	return geo2.Nearest(ctx, storeAdapter{s}, geo2.NewCalculator(geo2.Kilometers), geo2.Point{Lat: p.Lat, Lon: p.Lon, Alt: p.Alt}, names)
}
```

```go
// Fichier : release/geo/v2bridge.go
package geo

import (
	"context"

	geo2 "example.com/api/v2/geo"
)

// storeAdapter présente un Store v1 (sans contexte) comme un Store v2.
type storeAdapter struct{ s Store }

func (a storeAdapter) Get(_ context.Context, name string) (geo2.Point, error) {
	p, err := a.s.Get(name)
	return geo2.Point{Lat: p.Lat, Lon: p.Lon, Alt: p.Alt}, err
}

func (a storeAdapter) Put(_ context.Context, name string, p geo2.Point) error {
	return a.s.Put(name, Point{Lat: p.Lat, Lon: p.Lon, Alt: p.Alt})
}
```

```go
// Fichier : both/both.go
// Package both : un programme qui utilise les deux versions en même temps, pendant sa migration.
package both

import (
	geo1 "example.com/api/release/geo"
	geo2 "example.com/api/v2/geo"
)

// Une partie du code est déjà passée à v2 (calcul en miles) ; une autre utilise encore v1.
func Mixed() (float64, float64) {
	a, b := geo1.Point{Lat: 48.85, Lon: 2.35}, geo1.Point{Lat: 45.76, Lon: 4.84}
	km := geo1.Distance(a, b)
	mi := geo2.NewCalculator(geo2.Miles).Distance(geo2.Point{Lat: 48.85, Lon: 2.35}, geo2.Point{Lat: 45.76, Lon: 4.84})
	return km, mi
}
```

```go
// Fichier : both/both_test.go
package both

import (
	"fmt"
	"testing"
)

func TestMixed(t *testing.T) {
	km, mi := Mixed()
	fmt.Printf("v1 : %.1f km ; v2 : %.1f mi\n", km, mi)
	if km < 390 || km > 400 {
		t.Fatal(km)
	}
}
```

```
$ apidiff v11.api example.com/api/release/geo      # v1.1.0 (export écrit avec apidiff -w) contre v1.2.0
(rien : aucun changement d'API)
$ go test -v ./both/
v1 : 391.5 km ; v2 : 243.3 mi
$ go fix -diff ./clientr/
-	return geo.Nearest(s, geo.Point{Lat: 43.6, Lon: 1.44}, []string{"paris", "lyon"})
+	return geo.NearestContext(context.Background(), s, geo.Point{Lat: 43.6, Lon: 1.44}, []string{"paris", "lyon"})
```

1 et 2. **La v1 implémentée par la v2.** `NearestContext` appelle `geo2.Nearest`, à travers un adaptateur qui
présente un `Store` v1 (sans contexte) comme un `Store` v2. L'API de la v1 ne change **pas du tout** (`apidiff`
ne trouve rien entre v1.1.0 et v1.2.0), mais il n'y a plus qu'une implémentation à maintenir.

**Ce que `go fix` sait faire, et ce qu'il ne sait pas faire.** Il migre `Nearest` → `NearestContext` à
l'intérieur de la v1, grâce au `//go:fix inline` de v1.1.0. Il ne fait **pas** le pas suivant, vers la v2,
même avec `//go:fix inline` ajouté sur `NearestContext` (essayé : le résultat est identique, sans message).
Le corps de `NearestContext` utilise `storeAdapter`, un type **non exporté**, que le code de l'utilisateur ne
peut pas nommer : la recopie du corps est impossible, et `go fix` renonce en silence.

C'est la limite de l'outil : il migre les **délégations simples**, pas les changements de conception. Un
`Store` qui prend désormais un `ctx` demande de modifier le code de l'utilisateur, ses implémentations
comprises.

3. **Les deux versions dans le même programme**. `both` utilise v1 et v2 côte à côte : 391,5 km par la v1,
243,3 miles par la v2. C'est ce qui permet une migration **progressive** d'un gros programme, paquet par
paquet, au lieu d'un basculement d'un seul coup. Les types restent distincts (`geo1.Point` ≠ `geo2.Point`) : on
convertit aux frontières.

4. **Le guide de migration**, par exemple :

> **De `example.com/api` v1 à `example.com/api/v2`.**
> 1. `go fix ./...` sur la v1.2 : les appels à `Nearest` deviennent `NearestContext` (automatique).
> 2. Remplacer l'import `example.com/api/release/geo` par `example.com/api/v2/geo` (à la main, ou par
>    recherche et remplacement).
> 3. `NearestContext(ctx, s, p, noms)` devient `Nearest(ctx, s, geo.NewCalculator(geo.Kilometers), p, noms)`.
> 4. `Distance(a, b)` devient `geo.NewCalculator(geo.Kilometers).Distance(a, b)` ; `DistanceIn(a, b, u)`
>    devient `geo.NewCalculator(u).Distance(a, b)`.
> 5. **Vos implémentations de `Store`** prennent un `ctx context.Context` en premier paramètre de `Get` et de
>    `Put`. C'est le seul changement qui demande de la réflexion : utilisez `ctx` pour vos appels à la base.
> 6. `Delete` et `Deleter` n'existent plus : si votre `Store` sait supprimer, gardez votre propre méthode.
> 7. Les deux versions peuvent coexister : migrez paquet par paquet.

5. **La maintenance de la v1** : correctifs de sécurité et de bugs graves pendant une durée **annoncée** (par
exemple un an après la v2.0.0), pas de nouvelles fonctionnalités. La v1 n'étant plus qu'une couche sur la v2,
la plupart des corrections de la v2 lui profitent sans travail supplémentaire.

---

## Défi — Des contrats dictés par les clients

```go
// Fichier : e1/contract/contract.go
// Package contract : des contrats dictés par les clients. Chaque client déclare les champs de la
// réponse qu'il UTILISE et leur type ; l'outil les vérifie contre un serveur.
package contract

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strings"
)

// Contract : une attente d'un client.
type Contract struct {
	Client string            `json:"client"`
	Method string            `json:"method"`
	Path   string            `json:"path"`
	Fields map[string]string `json:"fields"` // chemin pointé → "string", "number", "bool", "object", "array"
}

// Violation : un champ dont un client dépend, absent ou d'un autre type.
type Violation struct {
	Client, Method, Path, Field, Want, Got string
}

func (v Violation) String() string {
	if v.Got == "absent" {
		return fmt.Sprintf("%s : %s %s : champ %q absent (attendu %s)", v.Client, v.Method, v.Path, v.Field, v.Want)
	}
	return fmt.Sprintf("%s : %s %s : champ %q de type %s, attendu %s", v.Client, v.Method, v.Path, v.Field, v.Got, v.Want)
}

// Check rejoue les contrats contre baseURL et renvoie toutes les violations.
func Check(c *http.Client, baseURL string, contracts []Contract) ([]Violation, error) {
	var out []Violation
	for _, ct := range contracts {
		req, err := http.NewRequest(ct.Method, baseURL+ct.Path, nil)
		if err != nil {
			return nil, err
		}
		resp, err := c.Do(req)
		if err != nil {
			return nil, err
		}
		var body any
		err = json.NewDecoder(resp.Body).Decode(&body)
		resp.Body.Close()
		if err != nil {
			return nil, fmt.Errorf("%s %s : %w", ct.Method, ct.Path, err)
		}
		for field, want := range ct.Fields {
			if got := typeAt(body, field); got != want {
				out = append(out, Violation{ct.Client, ct.Method, ct.Path, field, want, got})
			}
		}
	}
	return out, nil
}

// typeAt suit un chemin pointé (« customer.name ») et renvoie le type JSON trouvé, ou « absent ».
func typeAt(v any, path string) string {
	for part := range strings.SplitSeq(path, ".") {
		m, ok := v.(map[string]any)
		if !ok {
			return "absent"
		}
		if v, ok = m[part]; !ok {
			return "absent"
		}
	}
	switch v.(type) {
	case string:
		return "string"
	case float64:
		return "number"
	case bool:
		return "bool"
	case map[string]any:
		return "object"
	case []any:
		return "array"
	case nil:
		return "null"
	}
	return "inconnu"
}
```

```go
// Fichier : e1/contract/contract_test.go
package contract

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"
	"sort"
	"testing"
)

// Les contrats de trois clients (en vrai : des fichiers JSON publiés par chaque client).
const contractsJSON = `[
 {"client": "appli-mobile", "method": "GET", "path": "/orders/1", "fields": {"id": "number", "status": "string", "total": "number"}},
 {"client": "facturation", "method": "GET", "path": "/orders/1", "fields": {"id": "number", "total": "number", "customer.email": "string"}},
 {"client": "tableau-de-bord", "method": "GET", "path": "/orders/1", "fields": {"status": "string"}}
]`

func server(body string) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprint(w, body)
	}))
}

func TestServerChanges(t *testing.T) {
	var contracts []Contract
	if err := json.Unmarshal([]byte(contractsJSON), &contracts); err != nil {
		t.Fatal(err)
	}
	for _, tc := range []struct {
		change, body string
		ok           bool
	}{
		{"aujourd'hui", `{"id":1,"status":"paid","total":1250,"customer":{"email":"a@b.c"}}`, true},
		{"ajout d'un champ", `{"id":1,"status":"paid","total":1250,"currency":"EUR","customer":{"email":"a@b.c"}}`, true},
		{"renommage total → amount", `{"id":1,"status":"paid","amount":1250,"customer":{"email":"a@b.c"}}`, false},
		{"type de total changé", `{"id":1,"status":"paid","total":"12.50","customer":{"email":"a@b.c"}}`, false},
	} {
		srv := server(tc.body)
		v, err := Check(srv.Client(), srv.URL, contracts)
		srv.Close()
		if err != nil {
			t.Fatal(err)
		}
		lines := make([]string, len(v))
		for i, x := range v {
			lines[i] = x.String()
		}
		sort.Strings(lines)
		fmt.Printf("%-26s : %d violation(s)\n", tc.change, len(v))
		for _, l := range lines {
			fmt.Println("    " + l)
		}
		if (len(v) == 0) != tc.ok {
			t.Errorf("%s : %d violations", tc.change, len(v))
		}
	}
}
```

```
$ go test -v ./e1/contract/
aujourd'hui                : 0 violation(s)
ajout d'un champ           : 0 violation(s)
renommage total → amount   : 2 violation(s)
    appli-mobile : GET /orders/1 : champ "total" absent (attendu number)
    facturation : GET /orders/1 : champ "total" absent (attendu number)
type de total changé       : 2 violation(s)
    appli-mobile : GET /orders/1 : champ "total" de type string, attendu number
    facturation : GET /orders/1 : champ "total" de type string, attendu number
```

- **Un contrat ne déclare que ce que le client utilise.** Le tableau de bord ne lit que `status` : le
  renommage de `total` ne le concerne pas, et il n'apparaît pas dans les violations. Un test qui comparerait
  la réponse **entière** refuserait l'ajout de `currency`, inoffensif pour tout le monde.
- **Le message dit qui casse et pourquoi** (`facturation : GET /orders/1 : champ "total" absent`). L'équipe du
  serveur sait quelle équipe prévenir, et quelle transition prévoir (E5).
- **Contre un test de bout en bout** : il ne faut pas faire tourner les trois clients. Les contrats sont des
  données, rejouées en millisecondes dans la CI du **serveur**, avant tout déploiement.
- **Contre `apidiff`** : `apidiff` compare deux versions d'une API **Go**, sur les types et les signatures.
  Les contrats vérifient un format **sur le fil**, dont le compilateur ne sait rien.
- **Qui maintient les contrats ?** Chaque **client** maintient le sien, dans son dépôt, et le publie (dans un
  dépôt partagé ou un service de contrats). Le serveur ne peut pas deviner ce que ses clients utilisent. C'est
  tout l'intérêt de la méthode. Un contrat que personne ne met à jour finit par protéger des champs que plus
  personne ne lit : on lui donne une date, et on le supprime avec le client.

---

## Quiz

1. Le type de la fonction change : tous les appels compilent encore, mais un code qui l'utilise comme
   **valeur** (affectation à une variable de type fonction, passage en paramètre) ne compile plus.
2. Toutes les implémentations existantes, chez les utilisateurs, cessent de satisfaire l'interface. On ajoute
   une interface **optionnelle** séparée, découverte par assertion de type.
3. Oui, selon la promesse de Go. Pas pour les littéraux sans noms de champs (que `go vet` signale), ni pour
   `==` et les clés de map si le champ rend la struct non comparable (E1, changement 4).
4. Une interface séparée qu'un type **peut** implémenter pour offrir une capacité de plus, et que le code
   découvre par assertion de type : `io.WriterTo`, `io.ReaderFrom`, `http.Flusher`.
5. Un paragraphe `Deprecated:` dans le commentaire de documentation, avec le remplacement. `go doc`, `gopls`
   et `staticcheck` (SA1019) le signalent.
6. Il permet à `go fix` de remplacer chaque appel (ou usage d'une constante, d'un alias) par le corps de la
   fonction dépréciée, quand ce corps n'est qu'une délégation exprimable chez l'appelant.
7. Avec un nouveau chemin de module, terminé par `/v2` (`module example.com/api/v2`), une étiquette
   `v2.0.0`, et des imports qui incluent `/v2`.
8. Rien de visible : le champ renommé est inconnu et ignoré, et l'ancien vaut zéro (`Total: 0`). C'est une
   donnée fausse, en silence.
9. Il échoue dès que le serveur ajoute un champ, ce qui est le changement compatible par excellence. On le
   réserve aux entrées qu'on contrôle (configuration).
10. Les changements d'API Go (types, signatures, constantes, comparabilité) entre deux versions d'un paquet ou
    d'un module. Pas le comportement (E1 : la copie au lieu du pointeur), ni les formats sur le fil.
