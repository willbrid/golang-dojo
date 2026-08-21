# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Zéro-valeur utile

```go
type Config struct {
	Name    string            // ""      utilisable
	Port    int               // 0       utilisable
	Debug   bool              // false   utilisable
	Tags    []string          // nil     utilisable en LECTURE (len, range, append)
	Headers map[string]string // nil     LECTURE seulement — écrire PANIQUE
}

var c Config
fmt.Printf("%+v\n", c)
// {Name: Port:0 Debug:false Tags:[] Headers:map[]}
```

`%+v` affiche `[]` et `map[]` : **l'affichage ne distingue pas nil de vide**. Seul
`c.Headers == nil` le dit.

Les quatre premiers champs sont immédiatement utilisables. Le cinquième est un piège :
`c.Headers["x"] = "y"` panique. Trois conceptions possibles :

```go
// A. Un constructeur qui initialise
func NewConfig() *Config { return &Config{Headers: make(map[string]string)} }

// B. Une méthode qui initialise paresseusement (leçon 3)
func (c *Config) SetHeader(k, v string) {
	if c.Headers == nil {
		c.Headers = make(map[string]string)
	}
	c.Headers[k] = v
}

// C. Ne pas exposer la map du tout, et n'offrir que des accesseurs
```

**B est souvent la meilleure** : elle préserve la propriété « la zéro-valeur est utilisable »
sans imposer de constructeur. C'est ce que fait `strings.Builder`.

## E2 — Copie superficielle

```go
type Mixed struct {
	N int
	S []string
	M map[string]int
	P *int
}
a := Mixed{1, []string{"x"}, map[string]int{"k": 1}, new(int)}
b := a
b.N = 99          // indépendant
b.S[0] = "MOD"    // PARTAGÉ → a.S[0] change aussi
b.M["k"] = 99     // PARTAGÉ
*b.P = 99         // PARTAGÉ
```

Un seul champ sur quatre est réellement copié. **La copie d'une struct est superficielle : elle
s'arrête au premier niveau.** Slice, map et pointeur contiennent une adresse ; c'est l'adresse
qui est dupliquée, pas ce qu'elle désigne.

Corollaire : « passer une struct par valeur pour être sûr qu'elle ne sera pas modifiée » est
une **fausse sécurité** dès qu'elle contient une référence.

## E3 — Comparabilité

```go
type A struct{ X int; S string }   // comparable ✔ — clé de map possible
type B struct{ Tags []string }     // NON comparable ✘ — invalid map key type
type C struct{ Tags [3]string }    // comparable ✔ — un TABLEAU est comparable
```

Un tableau de taille fixe est comparable si son type d'élément l'est ; un slice ne l'est
jamais, parce que comparer deux slices supposerait de choisir entre comparer les descripteurs
et comparer les contenus — Go refuse d'arbitrer et interdit l'opération.

`A` et `C` peuvent servir de clés de map ; `B` non.

## E4 — Littéral positionnel

Avec `u := User{1, "Alice", "a@x.fr"}` puis ajout d'un champ **au milieu** du type :

```
./main.go:12:15: too few values in struct literal of type User
```

Si le champ est ajouté **à la fin**, le message est le même. Mais le cas vraiment dangereux est
l'ajout d'un champ **de même type** au milieu : le littéral compile alors parfaitement et
**affecte les valeurs aux mauvais champs**. Aucune erreur, un bug silencieux.

Avec un littéral nommé, l'ajout d'un champ ne casse rien : le nouveau champ prend sa
zéro-valeur. C'est la raison pratique de la règle, plus forte que l'argument de lisibilité.

`go vet` avec l'analyse `composites` signale les littéraux positionnels sur des types
**d'autres packages** — pas sur les siens.

## E5 — Balises

```go
type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name,omitempty"`
	Token string `json:"-"`
	Bad   string `jsom:"oops"`
}
```

`go vet` ne dit **rien** : `jsom` est une clé inconnue, mais syntaxiquement valide. Le champ
sera simplement sérialisé sous le nom `Bad`.

Ce que `go vet` détecte (analyse `structtag`) :
```
struct field tag `json:"id" xml:id` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
```
c'est-à-dire les **erreurs de syntaxe** — guillemets manquants, espaces mal placés — pas les
noms de clés erronés. Une faute de frappe dans le nom du sérialiseur passe donc entre les
mailles, et ne se voit qu'à l'exécution, en constatant que le JSON produit ne correspond pas.

## Exercice intermédiaire — `geometry`

```go
package main

import (
	"errors"
	"fmt"
	"math"
	"slices"
)

// epsilon : tolérance de comparaison des flottants.
// 1e-9 est adapté à des coordonnées de l'ordre de l'unité au million. Pour des
// coordonnées géographiques en degrés, ce serait environ 0,1 mm — suffisant.
// Une tolérance ABSOLUE devient inadaptée sur de très grandes valeurs : une
// version robuste utiliserait une tolérance relative.
const epsilon = 1e-9

type Point struct{ X, Y float64 }
type Segment struct{ A, B Point }
type Polygon struct{ Vertices []Point }
type Circle struct {
	Center Point
	Radius float64
}

var ErrTooFewVertices = errors.New("un polygone exige au moins 3 sommets")

func Distance(a, b Point) float64 {
	return math.Hypot(b.X-a.X, b.Y-a.Y) // Hypot évite le débordement de x²+y²
}

func SegmentLength(s Segment) float64 { return Distance(s.A, s.B) }

func PolygonPerimeter(p Polygon) (float64, error) {
	if len(p.Vertices) < 3 {
		return 0, fmt.Errorf("périmètre : %w (%d)", ErrTooFewVertices, len(p.Vertices))
	}
	total := 0.0
	for i, v := range p.Vertices {
		next := p.Vertices[(i+1)%len(p.Vertices)] // le modulo referme le polygone
		total += Distance(v, next)
	}
	return total, nil
}

// PolygonArea utilise la formule du lacet.
// CHOIX DOCUMENTÉ : on retourne la valeur ABSOLUE. L'aire signée porte
// l'orientation (négative en sens horaire), information utile en géométrie
// computationnelle — mais « aire » désigne une grandeur positive dans le
// langage courant, et l'API ne doit pas surprendre. Une fonction
// SignedArea séparée exposerait l'orientation à qui en a besoin.
func PolygonArea(p Polygon) (float64, error) {
	if len(p.Vertices) < 3 {
		return 0, fmt.Errorf("aire : %w (%d)", ErrTooFewVertices, len(p.Vertices))
	}
	sum := 0.0
	for i, v := range p.Vertices {
		next := p.Vertices[(i+1)%len(p.Vertices)]
		sum += v.X*next.Y - next.X*v.Y
	}
	return math.Abs(sum) / 2, nil
}

func BoundingBox(p Polygon) (lo, hi Point, err error) {
	if len(p.Vertices) == 0 {
		return Point{}, Point{}, errors.New("polygone vide")
	}
	lo, hi = p.Vertices[0], p.Vertices[0]
	for _, v := range p.Vertices[1:] {
		lo.X, lo.Y = math.Min(lo.X, v.X), math.Min(lo.Y, v.Y)
		hi.X, hi.Y = math.Max(hi.X, v.X), math.Max(hi.Y, v.Y)
	}
	return lo, hi, nil
}

// ClonePolygon produit une copie RÉELLEMENT indépendante.
// `q := p` ne suffirait pas : le slice Vertices serait partagé.
func ClonePolygon(p Polygon) Polygon {
	return Polygon{Vertices: slices.Clone(p.Vertices)}
}

// PolygonEqual compare sommet à sommet, avec tolérance.
// CHOIX DOCUMENTÉ : deux polygones décrivant la même forme mais commençant par
// un sommet différent sont considérés DIFFÉRENTS. Une égalité « à rotation
// près » serait plus juste géométriquement, mais coûte O(n²) et surprend
// l'appelant qui compare deux listes de points. Une fonction SameShape
// séparée conviendrait mieux à ce besoin.
func PolygonEqual(a, b Polygon) bool {
	if len(a.Vertices) != len(b.Vertices) {
		return false
	}
	for i := range a.Vertices {
		if !pointEqual(a.Vertices[i], b.Vertices[i]) {
			return false
		}
	}
	return true
}

func pointEqual(a, b Point) bool {
	return math.Abs(a.X-b.X) < epsilon && math.Abs(a.Y-b.Y) < epsilon
}
```

**Les cinq points de correction :**

1. **`math.Hypot` plutôt que `math.Sqrt(dx*dx + dy*dy)`.** Sur de très grandes coordonnées,
   `dx*dx` peut déborder vers `+Inf` alors que la distance, elle, est représentable. `Hypot`
   est conçue pour éviter ce cas. C'est un exemple de « la bibliothèque standard connaît un
   piège que je ne connais pas ».
2. **Le modulo `(i+1)%len` referme le polygone** sans cas particulier pour le dernier sommet.
3. **Aucune comparaison de flottants avec `==`.** `pointEqual` utilise une tolérance, et sa
   valeur est justifiée — une tolérance non justifiée est un nombre magique.
4. **`slices.Clone` dans `ClonePolygon`.** `q := p` copierait la struct mais partagerait le
   slice : la « copie » ne serait pas indépendante. C'est le cœur de l'exercice.
5. **Les choix ambigus sont documentés dans le commentaire de la fonction**, pas dans un
   fichier annexe. Le lecteur de l'API les voit avec `go doc`.

**Sur l'aire signée :** ce n'est ni un bug ni une information à jeter. Le signe indique le sens
de parcours des sommets, ce qui sert en géométrie computationnelle (détection d'orientation,
triangulation). La bonne conception est donc **deux fonctions** — `Area` positive et
`SignedArea` — plutôt qu'une fonction dont le signe surprend.

## Défi

**a) Copie profonde générale**

```go
type Level3 struct{ Points map[string][]Point }
type Level2 struct{ Inner Level3; Labels []string }
type Level1 struct{ Name string; Sub Level2 }

func DeepClone(l Level1) Level1 {
	out := l                                   // copie superficielle
	out.Sub.Labels = slices.Clone(l.Sub.Labels)
	out.Sub.Inner.Points = make(map[string][]Point, len(l.Sub.Inner.Points))
	for k, v := range l.Sub.Inner.Points {
		out.Sub.Inner.Points[k] = slices.Clone(v) // maps.Clone ne suffirait PAS :
	}                                             // il copierait les descripteurs
	return out
}
```

**`maps.Clone` est superficielle elle aussi** : elle copie les paires clé/valeur, donc les
descripteurs de slice. Sans le `slices.Clone` intérieur, les slices resteraient partagés. C'est
le piège que presque tout le monde manque.

**Le coût de maintenance :** chaque nouveau champ de référence ajouté à l'un des trois niveaux
exige une ligne supplémentaire dans `DeepClone`. Rien ne le rappelle — ni le compilateur, ni
`go vet`. Un champ oublié produit un partage silencieux qui se manifestera un jour, très loin
du lieu du bug.

| Alternative | Performance | Sécurité | Maintenance |
|---|---|---|---|
| À la main | excellente | exacte, mais l'oubli est invisible | mauvaise |
| Sérialisation/désérialisation (JSON, gob) | mauvaise (10 à 100×) | perd les champs non exportés, les cycles, les types non sérialisables | nulle |
| `reflect` générique | moyenne | gère tout, mais les cycles demandent un suivi | nulle |
| Génération de code (`go:generate`) | excellente | exacte, régénérée avec le type | faible |

**La réponse honnête dépend du contexte.** Sur un chemin chaud, la version à la main
accompagnée d'un **test qui vérifie l'indépendance** de chaque champ. Sur du code de
configuration appelé trois fois par jour, la sérialisation, dont la lenteur ne se voit pas. Sur
un type qui change souvent, la génération de code. Ce qu'il ne faut jamais faire : écrire la
version à la main **sans test** — c'est le seul choix qui échoue silencieusement.

**b) Alignement mémoire**

```go
type Pire struct {  b1 bool; i64 int64; b2 bool; i32 int32; s string; b3 bool; f float64 }
type Bonne struct { s string; i64 int64; f float64; i32 int32; b1, b2, b3 bool }
```

| | `unsafe.Sizeof` |
|---|---|
| `Pire` | 72 octets |
| `Bonne` | 56 octets |

Environ 22 % d'économie, obtenue uniquement en réordonnant. Sur un million d'instances : 72 Mo
contre 56 Mo, soit 16 Mo de bourrage évités.

**À partir de quand cela vaut-il la peine ?** Le critère n'est pas la taille absolue mais le
produit *taille du gain × nombre d'instances × pression sur le cache*. Ordres de grandeur :

- moins de 10 000 instances : jamais. Le gain est inférieur à la marge de bruit.
- 10⁵ à 10⁶ instances de structs allouées en flux continu : à considérer, surtout si le gain
  fait passer la struct sous une ligne de cache (64 octets).
- structures de données chaudes d'un serveur haute performance : oui, et l'outil
  `fieldalignment` de `golang.org/x/tools` le fait automatiquement.

Dans tous les cas : **mesurer d'abord**. Réordonner les champs rend le regroupement logique
moins lisible, ce qui a un coût réel de maintenance.

**c) Conception**

| Critère | Struct plate (20 champs) | Struct composée (5 champs) |
|---|---|---|
| Lisibilité | mauvaise au-delà de ~10 champs | bonne : la structure du domaine est visible |
| Copie | superficielle, comme l'autre | idem — l'imbrication par valeur ne change rien |
| Comparabilité | dépend des champs | idem |
| JSON | plat naturellement | imbriqué naturellement ; `,inline` n'existe pas en `encoding/json` — il faut de l'embedding |
| Évolutivité | chaque ajout allonge la liste | chaque ajout va dans le bon sous-groupe |
| Mémoire | légèrement meilleure (moins de bourrage entre groupes) | légèrement pire |

**Le contexte où la struct plate gagne** : quand elle représente une **ligne** — un
enregistrement de base de données, un CSV, un message de protocole. La forme plate correspond
alors exactement à la forme des données, et l'imbrication n'ajouterait qu'une traduction
inutile. C'est pourquoi les structs de la couche persistance sont souvent plates alors que
celles du domaine sont composées : elles ne modélisent pas la même chose.

## Réponses du quiz

1. Littéral nommé, littéral positionnel, `&T{…}` pour un pointeur, et `var x T` pour la
   zéro-valeur.
2. Il casse dès qu'un champ est ajouté ou déplacé — silencieusement si les types coïncident.
   Et il est impossible depuis un autre package dès qu'un champ n'est pas exporté.
3. Tous les champs, mais **superficiellement** : le descripteur du slice est copié, le tableau
   sous-jacent reste partagé.
4. Quand **tous** ses champs sont comparables.
5. Une struct non comparable ne peut pas être clé de map : le compilateur refuse.
6. `Billing Address` est **contenue** dans la struct (pas d'indirection, pas de `nil` possible)
   ; `Billing *Address` est référencée, peut valoir `nil` et être partagée entre plusieurs
   structs.
7. Une chaîne littérale accolée à un champ, ignorée par le compilateur, lue **à l'exécution**
   par réflexion — par `encoding/json`, `database/sql`, les validateurs.
8. **Non.** Elle compile parfaitement. `go vet` détecte les balises **syntaxiquement**
   malformées, pas les noms de clés erronés.
9. Il est lent (il utilise la réflexion), et sa notion d'égalité surprend : deux slices `nil`
   et vide y sont **différents**, deux `NaN` aussi. Il convient aux tests, pas à la logique
   métier.
10. Parce que chaque champ est **aligné** sur sa taille naturelle, ce qui insère du bourrage.
    Réordonner du plus grand au plus petit le minimise.
