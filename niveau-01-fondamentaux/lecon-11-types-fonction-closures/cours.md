# Leçon 11 — Types fonction et closures

## Objectifs

1. Traiter les fonctions comme des **valeurs** : les stocker, les passer, les retourner.
2. Déclarer des **types fonction** nommés et comprendre ce qu'ils apportent.
3. Maîtriser les **closures** : ce qu'elles capturent, où vit la variable capturée.
4. Reconnaître les patrons qui en découlent — fonctions d'ordre supérieur, décorateurs,
   options fonctionnelles.

## Explication

### Une fonction est une valeur comme une autre

```go
func add(a, b int) int { return a + b }

var op func(int, int) int   // zéro-valeur : nil
op = add
fmt.Println(op(2, 3))       // 5
fmt.Printf("%T\n", op)      // func(int, int) int
```

Le **type** d'une fonction est sa signature : paramètres et résultats, sans les noms.
`func(int, int) int` est un type à part entière. On peut donc en faire :

- une variable, comme ci-dessus ;
- un **paramètre** : `func Apply(nums []int, f func(int) int) []int` ;
- un **résultat** : `func Multiplier(n int) func(int) int` ;
- un **champ de struct** : `type Handler struct{ OnError func(error) }` — les structs sont au
  programme du niveau 2, mais l'usage se comprend dès maintenant ;
- un élément de slice ou une valeur de map : `map[string]func() error` — le remplaçant
  idiomatique du `switch` géant sur des noms de commandes.

Appeler une variable de fonction valant `nil` **panique**. Quand une fonction peut
légitimement être absente (un rappel optionnel), il faut tester :

```go
if h.OnError != nil {
	h.OnError(err)
}
```

### Fonctions anonymes

```go
// Déclarée puis appelée
f := func(a, b int) int { return a * b }
fmt.Println(f(3, 4))

// Appelée immédiatement
func() { fmt.Println("hop") }()

// Passée directement en argument
slices.SortFunc(people, func(a, b Person) int {
	return cmp.Compare(a.Age, b.Age)
})
```

Une fonction anonyme n'a pas de nom mais possède un type. C'est la forme la plus courante :
la bibliothèque standard en attend partout (`sort`, `slices`, `strings.TrimFunc`,
`http.HandlerFunc`, `filepath.WalkDir`…).

### Types fonction nommés

```go
type Validator func(string) error
type Middleware func(http.Handler) http.Handler
type Comparator[T any] func(a, b T) int
```

Déclarer un type pour une signature apporte trois choses :

1. **La lisibilité.** `func Chain(ms ...Middleware) Middleware` se lit ; la version dépliée
   ne se lit pas.
2. **La documentation.** Le type porte un nom et un commentaire, `go doc` l'affiche.
3. **Des méthodes.** Un type fonction peut avoir des méthodes — mécanisme au cœur de
   `net/http` :

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }
```

Cette astuce permet à une simple fonction de satisfaire une interface. On la retrouvera au
niveau 2 (interfaces) et au niveau 7 (HTTP) ; c'est l'un des idiomes les plus élégants de Go.

Attention : un type fonction nommé et sa signature sous-jacente restent **assignables** l'un
à l'autre. `var v Validator = maFonction` fonctionne si la signature correspond — pas besoin
de conversion.

### Closures : capture par référence

Une fonction anonyme **capture les variables de son environnement**, par référence et non
par copie.

```go
func counter() func() int {
	count := 0                  // survit au retour de counter !
	return func() int {
		count++                 // capture count par RÉFÉRENCE
		return count
	}
}

c := counter()
fmt.Println(c(), c(), c())      // 1 2 3
c2 := counter()
fmt.Println(c2())               // 1 — chaque closure a son propre count
```

`count` est déclaré dans `counter` mais référencé après son retour. Le compilateur détecte
qu'il **s'échappe** de la fonction et l'alloue sur le **tas** au lieu de la pile
(*escape analysis*, niveau 11). C'est automatique et sûr — contrairement au C, où retourner
un pointeur vers une variable locale est un bug grave.

Deux conséquences à retenir :

- **La capture est par référence.** Si la variable change après la création de la closure,
  la closure voit la nouvelle valeur.
- **Une closure alloue.** Elle porte un pointeur vers son environnement capturé. Ce n'est
  pas une raison de s'en priver, mais dans une boucle très chaude, cela se mesure (niveau 11).

Le piège classique de la variable de boucle a été corrigé en Go 1.22 (leçon 5), mais il
reste entier pour une variable déclarée **hors** de la boucle :

```go
var fns []func()
x := 0
for i := range 3 {
	x = i
	fns = append(fns, func() { fmt.Print(x) })  // capture LA MÊME variable x
}
for _, f := range fns { f() }                   // affiche 222, pas 012
```

### Trois patrons qui en découlent

**1. Fonction d'ordre supérieur** — une fonction qui en prend ou en retourne une autre.

```go
func Filter(nums []int, keep func(int) bool) []int
func Map(nums []int, f func(int) int) []int
```

Go n'a pas de `map`/`filter`/`reduce` intégrés sur les slices — c'est un choix assumé de
minimalisme. Depuis Go 1.23, le paquet `slices` et les itérateurs en fournissent une partie
(niveau 3).

**2. Fonction paramétrée** — une closure qui capture une configuration.

```go
func minLength(n int) Validator {
	return func(s string) error {
		if len([]rune(s)) < n {
			return fmt.Errorf("longueur < %d", n)
		}
		return nil
	}
}
```

**3. Décorateur** — une fonction qui en enveloppe une autre pour ajouter un comportement.

```go
func WithLogging(next func(string) error) func(string) error {
	return func(s string) error {
		fmt.Println("appel avec", s)
		err := next(s)
		fmt.Println("résultat :", err)
		return err
	}
}
```

C'est exactement la structure des **middlewares HTTP** du niveau 7. Reconnaître le patron
ici fait gagner beaucoup de temps plus tard.

### Aperçu : les options fonctionnelles

Go n'a pas d'arguments par défaut (leçon 9). L'idiome qui les remplace repose entièrement
sur les types fonction :

```go
type Option func(*Server)

func WithPort(p int) Option    { return func(s *Server) { s.port = p } }
func WithTimeout(d int) Option { return func(s *Server) { s.timeout = d } }

func NewServer(opts ...Option) *Server {
	s := &Server{port: 8080, timeout: 30} // valeurs par défaut
	for _, opt := range opts {
		opt(s)
	}
	return s
}

srv := NewServer(WithPort(9000))
```

Ce patron est partout dans l'écosystème Go (gRPC, OpenTelemetry, la plupart des clients de
bases de données). Il sera repris en détail au niveau 3, une fois les structs et les
pointeurs acquis. Il est présenté ici pour montrer **à quoi servent** les types fonction.

## Exemple

```go
package main

import (
	"fmt"
	"strings"
)

// Transform décrit une transformation de chaîne.
// Le type nommé rend les signatures suivantes lisibles.
type Transform func(string) string

// Compose retourne une Transform appliquant ts dans l'ordre.
// Le slice est copié à la construction : modifier ts après coup ne change pas
// le comportement de la transformation déjà construite.
func Compose(ts ...Transform) Transform {
	fns := make([]Transform, len(ts))
	copy(fns, ts)
	return func(s string) string {
		for _, t := range fns {
			s = t(s)
		}
		return s
	}
}

// replaceAll est une fonction PARAMÉTRÉE : elle capture old et new.
func replaceAll(old, new string) Transform {
	return func(s string) string { return strings.ReplaceAll(s, old, new) }
}

// withTrace est un DÉCORATEUR : il enveloppe une Transform.
func withTrace(name string, next Transform) Transform {
	return func(s string) string {
		out := next(s)
		fmt.Printf("  %-12s %q → %q\n", name, s, out)
		return out
	}
}

// Apply est une fonction d'ORDRE SUPÉRIEUR.
func Apply(items []string, t Transform) []string {
	out := make([]string, len(items)) // l'entrée n'est jamais modifiée
	for i, s := range items {
		out[i] = t(s)
	}
	return out
}

func main() {
	slug := Compose(
		withTrace("trim", strings.TrimSpace),
		withTrace("lower", strings.ToLower),
		withTrace("dashes", replaceAll(" ", "-")),
	)

	fmt.Println("résultat :", slug("  Bonjour Le Monde  "))

	// Une map de fonctions remplace un switch sur des noms de commandes.
	commands := map[string]Transform{
		"upper":   strings.ToUpper,
		"reverse": func(s string) string { r := []rune(s); for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 { r[i], r[j] = r[j], r[i] }; return string(r) },
	}
	if cmd, ok := commands["upper"]; ok {
		fmt.Println(Apply([]string{"go", "rust"}, cmd))
	}

	// Closure : chaque compteur a son propre état
	next := counter()
	fmt.Println(next(), next(), next())
}

func counter() func() int {
	n := 0
	return func() int { n++; return n }
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `type Transform func(string) string` | Sans ce type, `Compose(ts ...func(string) string) func(string) string` serait illisible. |
| `strings.TrimSpace` passée directement | Une fonction de la stdlib satisfait `Transform` sans conversion : les types fonction nommés restent assignables depuis leur signature. |
| `copy(fns, ts)` | Sans cette copie, la closure capturerait le slice de l'appelant : le modifier changerait la transformation **à distance**. Aliasing (leçon 6) déguisé en question de conception. |
| `replaceAll(old, new)` | Fonction paramétrée : la configuration est capturée, pas repassée à chaque appel. |
| `withTrace` | Décorateur : même signature en entrée et en sortie, donc empilable indéfiniment. |
| `map[string]Transform` | Remplace un `switch` sur des noms : extensible sans toucher au code existant. |
| `Apply` alloue un nouveau slice | Contrat clair : l'entrée n'est jamais modifiée. |

## Erreurs fréquentes

1. **Appeler une variable de fonction `nil`** → panique. Tester les rappels optionnels.
2. **Croire que la closure capture une copie** : elle capture la **variable**.
3. **Capturer une variable déclarée hors de la boucle** : toutes les closures voient la même.
4. **Croire que le correctif de Go 1.22 règle tous les cas de capture** : il ne concerne que la variable de boucle.
5. **Capturer un slice ou une map sans le copier** quand la valeur retournée est censée être stable.
6. **Empiler des closures sans nécessité** : trois niveaux d'indirection pour remplacer un `if` rendent le code plus difficile à déboguer, pas plus élégant.
7. **Oublier qu'une closure alloue** dans une boucle exécutée des millions de fois.
8. **Utiliser un type fonction là où une interface serait plus claire** : une fonction pour un comportement, une interface pour un objet à plusieurs comportements (niveau 2).

## Bonnes pratiques Go

- Déclarer un **type nommé** dès qu'une signature apparaît deux fois ou dépasse trois paramètres.
- Documenter ce que la fonction reçue est censée faire, et si elle peut être `nil`.
- Copier les slices et maps capturés quand la closure est retournée à un tiers.
- Préférer une fonction nommée à une fonction anonyme de plus de dix lignes.
- Une `map[string]func(...)` plutôt qu'un `switch` de vingt cas sur des chaînes.
- Réserver le décorateur aux comportements transversaux (journalisation, mesure, réessai),
  pas à la logique métier.
- Ne pas reproduire `map`/`filter`/`reduce` partout : en Go, une boucle explicite est
  souvent plus lisible qu'un enchaînement de fonctions d'ordre supérieur.

## Ce que je dois retenir

- Une fonction est une **valeur** : variable, paramètre, résultat, champ, entrée de map.
- Le **type** d'une fonction est sa signature ; un type nommé le rend lisible et peut
  porter des **méthodes**.
- Une **closure capture les variables par référence** ; elles s'échappent vers le tas.
- Chaque closure créée a son propre environnement — sauf si elle capture une variable
  partagée déclarée à l'extérieur.
- Appeler une fonction `nil` panique.
- Trois patrons en découlent : ordre supérieur, fonction paramétrée, décorateur — et les
  **options fonctionnelles**, qui remplacent les arguments par défaut absents de Go.

➡️ [Exercices](exercices.md)
