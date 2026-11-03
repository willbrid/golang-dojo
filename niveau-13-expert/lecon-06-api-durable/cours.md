# Module 6 — Conception d'API durable

> Le code d'un service se réécrit ; son API, elle, se traîne pendant des années, parce que d'autres en
> dépendent. Ces utilisateurs sont souvent inconnus : un autre service, un client mobile qu'on ne peut pas
> forcer à se mettre à jour, un module importé par des centaines de projets. Ce module traite de ce qui
> rend une API **durable** :
> - savoir exactement ce qui la casse ;
> - la faire évoluer sans casser ;
> - retirer proprement ce qui doit disparaître ;
> - assumer une version majeure quand elle est inévitable.
>
> Il le fait pour une bibliothèque Go, avec l'outil `apidiff` et `go fix`, et pour une API sur le fil (JSON),
> en mesurant ce que voit un ancien client.

## Objectifs

- Savoir quels changements d'une API Go sont compatibles, et les vérifier avec `apidiff`.
- Concevoir une API qui pourra évoluer : nouvelles fonctions, interfaces optionnelles, options.
- Déprécier avec la convention `Deprecated:`, et faire migrer les utilisateurs par `go fix`.
- Décider d'une version majeure, et la publier sous un nouveau chemin de module.
- Faire évoluer un format d'échange (JSON) sans casser les anciens clients.

## Explication

### Ce qui casse une API Go

La promesse de compatibilité de Go (go.dev/doc/go1compat) dit ce qu'une évolution compatible a le droit de
faire. La même règle vaut pour un module en version v1 ou plus (N2 L9) : une version mineure ou corrective ne
doit pas casser un programme qui compilait.

L'exemple part d'un paquet `geo` v1.0.0. Une version candidate y apporte des changements qui semblent
anodins. L'outil `apidiff` (`golang.org/x/exp/cmd/apidiff`) les classe :

```
Incompatible changes:
- Distance: changed from func(Point, Point) float64 to func(Point, Point, ...Unit) float64
- Nearest: changed from func(Store, Point, []string) (string, error) to func(context.Context, Store, Point, []string) (string, error)
- Store.Delete: added
Compatible changes:
- DistanceIn: added
- Kilometers: added
- Miles: added
- Point.Alt: added
- Unit: added
```

Puis un **client** écrit pour v1, du code ordinaire, est compilé contre la candidate :

```
client2/client.go:19:43: cannot use geo.Distance (value of type func(a geo.Point, b geo.Point, unit ...geo.Unit) float64) as func(a geo.Point, b geo.Point) float64 value in variable declaration
client2/client.go:22:37: too few values in struct literal of type geo.Point
client2/client.go:23:57: not enough arguments in call to geo.Nearest
```

| Changement | Qui casse | Compatible ? |
|---|---|---|
| ajouter une fonction, un type, une constante | personne | oui |
| ajouter un **paramètre variadique** | qui utilise la fonction comme **valeur** (`var f func(a, b Point) float64 = geo.Distance`) | **non**, même si tous les appels compilent encore |
| ajouter un paramètre (`ctx`) | tous les appelants | non |
| ajouter une **méthode à une interface** | tous ceux qui l'**implémentent** | non |
| ajouter un champ à une struct | ceux qui écrivent des littéraux **sans noms de champs** (`geo.Point{45.76, 4.84}`) | oui, selon la promesse de Go (et `go vet` signale ces littéraux) |
| renommer, retirer, changer un type | tout le monde | non |

Le cas du champ est instructif. `apidiff` le classe compatible, et la promesse de Go l'autorise. Pourtant, le
client qui avait écrit `geo.Point{45.76, 4.84}` ne compile plus. `go vet` l'avait prévenu dès v1 (« struct
literal uses unkeyed fields »). La compatibilité est un **contrat** à deux parties : l'auteur ne casse pas ce
que la règle protège, et l'utilisateur n'écrit pas ce qu'elle ne protège pas.

Au-delà de la signature, **tout comportement observable** finit par être utilisé par quelqu'un : l'ordre
d'une liste, le texte d'une erreur, une durée, un champ toujours rempli. Changer l'un d'eux est un changement
d'API, même si `apidiff` ne dit rien.

### Évoluer sans casser

La version v1.1.0 de l'exemple apporte les **mêmes** fonctionnalités que la candidate, sans rien casser.
`apidiff` ne trouve que des ajouts :

| Besoin | La candidate (incompatible) | La v1.1.0 (compatible) |
|---|---|---|
| une distance en miles | paramètre variadique sur `Distance` | une fonction **nouvelle**, `DistanceIn(a, b, unit)` |
| l'annulation | `ctx` ajouté à `Nearest` | `NearestContext(ctx, …)`, et `Nearest` qui l'appelle avec `context.Background()` |
| supprimer un point | `Delete` ajouté à l'interface `Store` | une interface **optionnelle** `Deleter`, et `geo.Delete(s, nom)` qui vérifie par assertion de type si `s` sait le faire |

Ce sont les trois techniques de base. On ajoute au lieu de modifier. Les capacités nouvelles passent par des
interfaces **optionnelles**, que la bibliothèque standard utilise partout : `io.WriterTo`, `http.Flusher`,
`io.ReaderFrom` (module 2).

Et on prévoit l'évolution **dès la première version** :

- **N'exporter que le nécessaire.** Ce qui n'est pas exporté peut changer librement, et `internal/` aussi
  (N10 L7).
- **Accepter des interfaces petites, renvoyer des types concrets** (N10 L4). On ajoute une méthode à une
  struct sans rien casser, pas à une interface.
- **Prévoir la place des paramètres futurs** : une struct d'options (`Options{…}`, à laquelle on ajoute des
  champs), ou des options fonctionnelles (`New(addr, WithTimeout(…))`).
- **`context.Context` dès le début** pour toute fonction qui fait des E/S. L'ajouter plus tard oblige à
  doubler la fonction.
- **Des erreurs comparables** (`ErrNotFound`, `errors.Is`) plutôt qu'un texte que les appelants
  analyseraient.

### Déprécier

Ce qui doit disparaître est d'abord **déprécié**. Selon la convention officielle, un paragraphe du
commentaire de documentation commence par `Deprecated:` et dit quoi utiliser à la place. Les outils le
reconnaissent :

```
$ staticcheck ./clientr/
clientr/client.go:23:9: example.com/api/release/geo.Nearest is deprecated: utiliser NearestContext, qui accepte l'annulation. Nearest sera retirée en v2.  (SA1019)
```

Et quand la fonction dépréciée n'est qu'une **délégation** à la nouvelle, la directive `//go:fix inline`
(N5 L8) permet à `go fix` de **réécrire le code des utilisateurs** :

```
$ go fix -diff ./clientr/
+import "context"
…
-	return geo.Nearest(s, geo.Point{Lat: 43.6, Lon: 1.44}, []string{"paris", "lyon"})
+	return geo.NearestContext(context.Background(), s, geo.Point{Lat: 43.6, Lon: 1.44}, []string{"paris", "lyon"})
```

La migration se fait sans aucune erreur humaine, et l'import est ajouté. C'est la meilleure façon de préparer
une version majeure : quand elle sortira, les utilisateurs auront déjà quitté l'ancienne fonction.

Une dépréciation a un **calendrier**, annoncé dans les notes de version (N12 L1). Une fonction dépréciée
reste au moins une version mineure, ou plusieurs mois, avant de disparaître dans une majeure. Elle ne
disparaît jamais dans une mineure.

### La version majeure

Quand un changement incompatible est vraiment nécessaire (une conception à refaire, plusieurs dépréciations à
retirer), on publie une **version majeure**. En Go, elle a un **autre chemin de module** : `module
example.com/api/v2`, et `import "example.com/api/v2/geo"` (N2 L9). Les deux versions coexistent, dans le même
programme si besoin, et chaque utilisateur migre à son rythme.

Une majeure a un coût **pour chaque utilisateur** : il doit changer ses imports et son code. On la réserve aux
cas où les techniques précédentes ne suffisent plus. On la prépare par des dépréciations et `go fix`. On
continue de publier des correctifs de sécurité pour la v1 pendant un temps annoncé.

`apidiff` en CI, contre la dernière étiquette publiée, empêche de publier par erreur une incompatibilité dans
une mineure : la CI échoue, et l'on choisit entre une nouvelle fonction et une majeure.

### Les API sur le fil

Pour une API HTTP/JSON (ou gRPC, N7 L12), le « client » est un programme qu'on ne recompile pas. Mesuré dans
l'exemple, un ancien client qui décode les réponses d'un serveur plus récent :

| Changement côté serveur | Client tolérant (défaut d'`encoding/json`) | Client strict (`DisallowUnknownFields`) |
|---|---|---|
| champ **ajouté** (`currency`) | fonctionne | **erreur** : `unknown field "currency"` |
| champ **renommé** (`total` → `amount`) | **`Total: 0`, aucune erreur** | erreur |
| type **changé** (`1250` → `"12.50"`) | erreur | erreur |
| valeur d'énumération **ajoutée** (`refunded`) | décodé, sans erreur | décodé, sans erreur |

- **Le renommage est le pire** : un client tolérant ne voit pas d'erreur, et affiche un total de 0. Une donnée
  fausse en silence vaut moins qu'une erreur.
- **Un client strict** est sûr, mais il casse au **moindre** ajout. Pour un client qui lit les réponses d'un
  serveur qui évolue, c'est le mauvais réglage. On le réserve aux **entrées** qu'on contrôle, comme les
  fichiers de configuration (N12 L5).
- **Une valeur d'énumération nouvelle** passe le décodage, mais le `switch` du client tombe dans un cas
  inattendu. Un client bien écrit a un cas « inconnu » qui ne plante pas.

D'où les règles d'un format durable :
- **on ajoute, on ne renomme jamais, on ne change jamais un type** ;
- **un champ nouveau est optionnel**, et son absence a un sens (ancien client, ancien serveur) ;
- **les clients tolèrent l'inconnu** (champs, valeurs d'énumération), et les serveurs produisent exactement ce
  qu'ils documentent ;
- **en gRPC, un numéro de champ n'est jamais réutilisé**, même après la suppression du champ.

Pour un vrai changement de sens, on ajoute un **nouveau champ** (`amount_cents`) à côté de l'ancien pendant la
transition, ou une nouvelle version de l'API (`/v2/orders`).

## Exemple

Un module avec la v1.0.0 de `geo`, la version candidate (incompatible), la v1.1.0 (compatible), deux clients
et le test du format JSON.

```go.mod
// Fichier : go.mod
module example.com/api

go 1.27
```

```go
// Fichier : v1/geo/geo.go
// Package geo : la version v1.0.0 publiée.
package geo

import "errors"

// Point est un point en coordonnées géographiques (degrés).
type Point struct {
	Lat, Lon float64
}

// Store mémorise des points nommés.
type Store interface {
	Get(name string) (Point, error)
	Put(name string, p Point) error
}

var ErrNotFound = errors.New("geo : point inconnu")

// Distance renvoie la distance en kilomètres entre deux points.
func Distance(a, b Point) float64 { return haversine(a, b) }

// Nearest renvoie le nom du point de s le plus proche de p parmi names.
func Nearest(s Store, p Point, names []string) (string, error) {
	best, bestD := "", -1.0
	for _, n := range names {
		q, err := s.Get(n)
		if err != nil {
			return "", err
		}
		if d := Distance(p, q); bestD < 0 || d < bestD {
			best, bestD = n, d
		}
	}
	return best, nil
}
```

```go
// Fichier : v1/geo/haversine.go
package geo

import "math"

func haversine(a, b Point) float64 {
	const r = 6371.0
	rad := math.Pi / 180
	dLat, dLon := (b.Lat-a.Lat)*rad, (b.Lon-a.Lon)*rad
	h := math.Sin(dLat/2)*math.Sin(dLat/2) + math.Cos(a.Lat*rad)*math.Cos(b.Lat*rad)*math.Sin(dLon/2)*math.Sin(dLon/2)
	return 2 * r * math.Asin(math.Sqrt(h))
}
```

```go
// Fichier : candidate/geo/geo.go
// Package geo : une version candidate, avec plusieurs changements à classer.
package geo

import (
	"context"
	"errors"
)

// Point : un champ AJOUTÉ (l'altitude).
type Point struct {
	Lat, Lon float64
	Alt      float64 // mètres
}

// Store : une méthode AJOUTÉE à l'interface.
type Store interface {
	Get(name string) (Point, error)
	Put(name string, p Point) error
	Delete(name string) error
}

var ErrNotFound = errors.New("geo : point inconnu")

// Unit : un type AJOUTÉ.
type Unit int

const (
	Kilometers Unit = iota
	Miles
)

// Distance : un paramètre variadique AJOUTÉ, « compatible » en apparence.
func Distance(a, b Point, unit ...Unit) float64 {
	d := haversine(a, b)
	if len(unit) > 0 && unit[0] == Miles {
		return d / 1.609344
	}
	return d
}

// Nearest : un premier paramètre context.Context AJOUTÉ.
func Nearest(ctx context.Context, s Store, p Point, names []string) (string, error) {
	best, bestD := "", -1.0
	for _, n := range names {
		if err := ctx.Err(); err != nil {
			return "", err
		}
		q, err := s.Get(n)
		if err != nil {
			return "", err
		}
		if d := Distance(p, q); bestD < 0 || d < bestD {
			best, bestD = n, d
		}
	}
	return best, nil
}

// DistanceIn : une fonction AJOUTÉE.
func DistanceIn(a, b Point, u Unit) float64 { return Distance(a, b, u) }
```

```go
// Fichier : candidate/geo/haversine.go
package geo

import "math"

func haversine(a, b Point) float64 {
	const r = 6371.0
	rad := math.Pi / 180
	dLat, dLon := (b.Lat-a.Lat)*rad, (b.Lon-a.Lon)*rad
	h := math.Sin(dLat/2)*math.Sin(dLat/2) + math.Cos(a.Lat*rad)*math.Cos(b.Lat*rad)*math.Sin(dLon/2)*math.Sin(dLon/2)
	return 2 * r * math.Asin(math.Sqrt(h))
}
```

```go
// Fichier : release/geo/geo.go
// Package geo : la version v1.1.0, compatible avec v1.0.0.
package geo

import (
	"context"
	"errors"
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

// NearestContext est Nearest avec un contexte (v1.1.0).
func NearestContext(ctx context.Context, s Store, p Point, names []string) (string, error) {
	best, bestD := "", -1.0
	for _, n := range names {
		if err := ctx.Err(); err != nil {
			return "", err
		}
		q, err := s.Get(n)
		if err != nil {
			return "", err
		}
		if d := Distance(p, q); bestD < 0 || d < bestD {
			best, bestD = n, d
		}
	}
	return best, nil
}
```

```go
// Fichier : release/geo/haversine.go
package geo

import "math"

func haversine(a, b Point) float64 {
	const r = 6371.0
	rad := math.Pi / 180
	dLat, dLon := (b.Lat-a.Lat)*rad, (b.Lon-a.Lon)*rad
	h := math.Sin(dLat/2)*math.Sin(dLat/2) + math.Cos(a.Lat*rad)*math.Cos(b.Lat*rad)*math.Sin(dLon/2)*math.Sin(dLon/2)
	return 2 * r * math.Asin(math.Sqrt(h))
}
```

```go
// Fichier : client/client.go
// Package client : un utilisateur de geo v1, écrit comme on écrit du code ordinaire.
package client

import "example.com/api/v1/geo"

// memStore : une implémentation de geo.Store, écrite par l'utilisateur.
type memStore map[string]geo.Point

func (m memStore) Get(n string) (geo.Point, error) {
	p, ok := m[n]
	if !ok {
		return geo.Point{}, geo.ErrNotFound
	}
	return p, nil
}

func (m memStore) Put(n string, p geo.Point) error { m[n] = p; return nil }

var metric func(a, b geo.Point) float64 = geo.Distance // une valeur de fonction

func Closest() (string, error) {
	s := memStore{"paris": {48.85, 2.35}, "lyon": geo.Point{45.76, 4.84}} // littéraux sans noms de champs
	return geo.Nearest(s, geo.Point{Lat: 43.6, Lon: 1.44}, []string{"paris", "lyon"})
}
```

```go
// Fichier : clientr/client.go
// Package client : un utilisateur de geo v1, écrit comme on écrit du code ordinaire.
package clientr

import "example.com/api/release/geo"

// memStore : une implémentation de geo.Store, écrite par l'utilisateur.
type memStore map[string]geo.Point

func (m memStore) Get(n string) (geo.Point, error) {
	p, ok := m[n]
	if !ok {
		return geo.Point{}, geo.ErrNotFound
	}
	return p, nil
}

func (m memStore) Put(n string, p geo.Point) error { m[n] = p; return nil }

var metric func(a, b geo.Point) float64 = geo.Distance // une valeur de fonction

func Closest() (string, error) {
	s := memStore{"paris": {Lat: 48.85, Lon: 2.35}, "lyon": geo.Point{Lat: 45.76, Lon: 4.84}}
	return geo.Nearest(s, geo.Point{Lat: 43.6, Lon: 1.44}, []string{"paris", "lyon"})
}
```

```go
// Fichier : wire/doc.go
// Package wire : la compatibilité d'un format d'échange.
package wire
```

```go
// Fichier : wire/wire_test.go
package wire

import (
	"bytes"
	"encoding/json"
	"fmt"
	"testing"
)

// Ce que l'ANCIEN client sait décoder (v1 de l'API HTTP).
type orderV1 struct {
	ID     int    `json:"id"`
	Status string `json:"status"` // "pending" ou "paid"
	Total  int    `json:"total"`
}

func decode(body string, strict bool) (orderV1, error) {
	var o orderV1
	dec := json.NewDecoder(bytes.NewReader([]byte(body)))
	if strict {
		dec.DisallowUnknownFields()
	}
	err := dec.Decode(&o)
	return o, err
}

// Ce que le serveur RÉCENT renvoie, selon le changement fait.
func TestOldClientNewServer(t *testing.T) {
	for _, tc := range []struct{ change, body string }{
		{"aucun", `{"id":1,"status":"paid","total":1250}`},
		{"champ ajouté", `{"id":1,"status":"paid","total":1250,"currency":"EUR"}`},
		{"champ renommé", `{"id":1,"status":"paid","amount":1250}`},
		{"type changé", `{"id":1,"status":"paid","total":"12.50"}`},
		{"valeur d'énumération ajoutée", `{"id":1,"status":"refunded","total":1250}`},
	} {
		for _, strict := range []bool{false, true} {
			o, err := decode(tc.body, strict)
			fmt.Printf("%-30s strict=%-5v → %+v, erreur : %v\n", tc.change, strict, o, err)
		}
	}
}
```

(`clientr` est le client de `client`, recompilé contre la v1.1.0, avec des littéraux nommés. Compilé tel quel,
il échoue : `too few values in struct literal of type geo.Point`, à cause du champ `Alt`.)

```
$ apidiff example.com/api/v1/geo example.com/api/candidate/geo
Incompatible changes:
- Distance: changed from func(Point, Point) float64 to func(Point, Point, ...Unit) float64
- Nearest: changed from func(Store, Point, []string) (string, error) to func(context.Context, Store, Point, []string) (string, error)
- Store.Delete: added
Compatible changes:
- DistanceIn: added
- Kilometers: added
- Miles: added
- Point.Alt: added
- Unit: added
$ apidiff example.com/api/v1/geo example.com/api/release/geo
Compatible changes:
- Delete: added
- Deleter: added
- DistanceIn: added
- ErrNotSupported: added
- Kilometers: added
- Miles: added
- NearestContext: added
- Point.Alt: added
- Unit: added
$ go vet ./client/
client/client.go:22:25: example.com/api/v1/geo.Point struct literal uses unkeyed fields
client/client.go:22:48: example.com/api/v1/geo.Point struct literal uses unkeyed fields
$ go test -v ./wire/
aucun                          strict=false → {ID:1 Status:paid Total:1250}, erreur : <nil>
aucun                          strict=true  → {ID:1 Status:paid Total:1250}, erreur : <nil>
champ ajouté                   strict=false → {ID:1 Status:paid Total:1250}, erreur : <nil>
champ ajouté                   strict=true  → {ID:1 Status:paid Total:1250}, erreur : json: unknown field "currency"
champ renommé                  strict=false → {ID:1 Status:paid Total:0}, erreur : <nil>
champ renommé                  strict=true  → {ID:1 Status:paid Total:0}, erreur : json: unknown field "amount"
type changé                    strict=false → {ID:1 Status:paid Total:0}, erreur : json: cannot unmarshal string into Go struct field orderV1.total of type int
type changé                    strict=true  → {ID:1 Status:paid Total:0}, erreur : json: cannot unmarshal string into Go struct field orderV1.total of type int
valeur d'énumération ajoutée   strict=false → {ID:1 Status:refunded Total:1250}, erreur : <nil>
valeur d'énumération ajoutée   strict=true  → {ID:1 Status:refunded Total:1250}, erreur : <nil>
```

## Explication du code

| Élément | Rôle |
|---|---|
| `v1/geo`, `candidate/geo`, `release/geo` | Trois versions du même paquet, dans un seul module, pour que `apidiff` et le compilateur les comparent. En vrai, on compare deux étiquettes Git. |
| `type Deleter interface{ Delete(string) error }` | Une capacité optionnelle : un `Store` existant reste un `Store`, et ceux qui savent supprimer le disent en implémentant `Deleter`. |
| `geo.Delete(s, name)` et `s.(Deleter)` | L'assertion de type découvre la capacité à l'exécution ; sans elle, `ErrNotSupported`. |
| `Deprecated:` | Un paragraphe du commentaire de documentation, au format reconnu par `go doc`, `gopls` et `staticcheck` (SA1019). |
| `//go:fix inline` | Le corps de `Nearest` est une délégation : `go fix` peut le recopier à la place de chaque appel. |
| `var metric … = geo.Distance` dans le client | La façon la plus discrète de dépendre de la **signature exacte** d'une fonction. |
| `DisallowUnknownFields` | Le client strict du test : refuse tout champ qu'il ne connaît pas. |

## Erreurs fréquentes

1. **Ajouter un paramètre** (même variadique) à une fonction publiée.
2. **Ajouter une méthode à une interface publiée** dans une version mineure.
3. **Exporter des types et des champs « au cas où »** : chacun devient une promesse.
4. **Supprimer ou renommer** sans dépréciation préalable.
5. **Une dépréciation sans remplacement**, ou sans calendrier.
6. **Une version majeure sans `/v2` dans le chemin du module** (N2 L9).
7. **Renommer un champ JSON**, ou changer son type, en croyant que les clients suivront.
8. **Des clients stricts** sur des réponses qui évoluent ; des `switch` sans cas pour les valeurs inconnues.
9. **Réutiliser un numéro de champ protobuf.**

## Bonnes pratiques Go

- Le minimum exporté ; `internal/` pour le reste ; des types concrets en retour.
- Ajouter des fonctions plutôt que modifier des signatures ; interfaces optionnelles pour les nouvelles
  capacités ; options extensibles.
- `context.Context` et des erreurs comparables dès la première version.
- `Deprecated:` avec le remplacement et le calendrier ; `//go:fix inline` quand la migration est mécanique.
- `apidiff` en CI contre la dernière étiquette ; une majeure seulement quand c'est inévitable, préparée par des
  dépréciations.
- Sur le fil : ajouts seulement, champs optionnels, clients tolérants, numéros de champs jamais réutilisés.

## Ce que je dois retenir

- Un paramètre variadique ajouté, un `ctx` ajouté, une méthode d'interface ajoutée : trois incompatibilités
  qu'`apidiff` détecte.
- Un champ ajouté est compatible selon Go, mais casse les littéraux sans noms, que `go vet` signale.
- Évoluer, c'est ajouter : nouvelles fonctions, interfaces optionnelles.
- `Deprecated:` et `//go:fix inline` : `go fix` migre le code des utilisateurs.
- En JSON, un champ renommé donne une donnée fausse en silence : on ajoute, on ne renomme jamais.

---

➡️ [Exercices](exercices.md)
