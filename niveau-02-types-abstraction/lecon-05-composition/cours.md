# Leçon 5 — Composition et embedding

## Objectifs

1. Utiliser l'**embedding** de structs et comprendre la **promotion** des champs et méthodes.
2. Savoir précisément en quoi ce n'est **pas** de l'héritage, et ce que cela change.
3. Composer des interfaces à partir d'interfaces plus petites.
4. Reconnaître les patrons qui en découlent : décorateur, interface partiellement implémentée.

## Explication

### Le champ anonyme

```go
type Animal struct{ Name string }

func (a Animal) Speak() string { return a.Name + " fait un bruit" }

type Dog struct {
	Animal        // champ ANONYME : embedding
	Breed  string
}
```

Un champ déclaré **sans nom** est *embarqué*. Son type devient implicitement le nom du champ :

```go
d := Dog{Animal: Animal{Name: "Rex"}, Breed: "berger"}

fmt.Println(d.Name)        // "Rex"        — champ PROMU
fmt.Println(d.Animal.Name) // "Rex"        — accès explicite, toujours possible
fmt.Println(d.Speak())     // méthode PROMUE
```

La **promotion** rend les champs et méthodes du type embarqué accessibles directement sur le
type extérieur. Le compilateur réécrit `d.Name` en `d.Animal.Name`.

On peut embarquer :
- une struct : `Animal` ;
- un pointeur vers une struct : `*Animal` — utile quand le composant est optionnel ou partagé ;
- un type non-struct : `type Celsius float64` ;
- une **interface** (voir plus bas).

### Ce n'est pas de l'héritage

Trois différences décisives.

**1. Pas de sous-typage.**
```go
var a Animal = d    // ERREUR : cannot use d (Dog) as Animal value
```
Un `Dog` n'**est pas** un `Animal`. Il en *contient* un. Pour les rendre interchangeables, il
faut une **interface** que les deux satisfont (leçon 4).

**2. Pas de dispatch virtuel.**
```go
func (d Dog) Speak() string { return d.Name + " aboie" }

d.Speak()         // "Rex aboie"        — la méthode extérieure MASQUE la promue
d.Animal.Speak()  // "Rex fait un bruit" — toujours accessible par son nom
```
Mais surtout :
```go
func (a Animal) Describe() string { return "je dis : " + a.Speak() }

d.Describe()      // "je dis : Rex fait un bruit"  ← PAS "Rex aboie"
```
Le code d'`Animal` appelle **toujours** la méthode d'`Animal`. Il ne connaît pas `Dog` et ne
peut pas l'appeler. C'est l'inverse exact de l'héritage, où `Describe` appellerait la version
redéfinie.

Cette différence élimine toute une classe de bugs — notamment le classique « le constructeur
de la classe mère appelle une méthode redéfinie qui utilise un champ pas encore initialisé ».
Elle a un prix : le patron *template method* n'existe pas tel quel en Go. On l'obtient en
passant explicitement une fonction ou une interface.

**3. La promotion est résolue à la compilation, par profondeur.**
Le champ ou la méthode de **moindre profondeur** gagne. À profondeur égale entre deux types
embarqués, il y a **ambiguïté** :

```go
type A struct{ Name string }
type B struct{ Name string }
type C struct{ A; B }

c := C{}
c.Name = "x"     // ERREUR : ambiguous selector c.Name
c.A.Name = "x"   // OK — il faut désambiguïser
```

Le compilateur ne choisit pas à notre place. C'est une bonne nouvelle : l'ambiguïté est
détectée, pas silencieusement arbitrée.

### Embedding d'interfaces

Une interface peut en embarquer d'autres :

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }

type ReadWriter interface {
	Reader
	Writer
}
```

C'est exactement ainsi que sont construits `io.ReadWriter`, `io.ReadCloser`,
`io.ReadWriteCloser`. La bibliothèque standard définit une poignée d'interfaces à une méthode
et les **combine** au besoin, plutôt que d'en écrire de grosses. C'est la philosophie
générale de Go appliquée aux contrats.

### Embarquer une interface dans une struct

Technique moins connue, et très utile :

```go
type LoggingStore struct {
	Store            // INTERFACE embarquée
	logger *log.Logger
}

func (l LoggingStore) Get(id string) ([]byte, error) {
	l.logger.Printf("get %s", id)
	return l.Store.Get(id)   // délégation explicite
}
```

`LoggingStore` satisfait `Store` **entièrement** : `Get` est redéfinie, toutes les autres
méthodes sont promues depuis le champ embarqué. On décore une méthode sans avoir à écrire de
délégation pour les vingt autres.

Deux usages :
- le **décorateur** (journalisation, mesure, cache, réessai) ;
- les **faux de test partiels** : on embarque l'interface sans la remplir, et on n'implémente
  que la méthode utilisée par le test. Les autres existent mais paniquent si on les appelle —
  ce qui est souvent exactement ce qu'on veut, puisque le test ne devrait pas les appeler.

Le piège : si le champ embarqué est `nil` et qu'une méthode **non redéfinie** est appelée,
c'est une panique `nil pointer dereference` — sans indication claire de la cause.

### Composition « à plat » : le choix par défaut

L'embedding n'est pas obligatoire. Un champ **nommé** est souvent plus clair :

```go
type Server struct {
	logger *slog.Logger    // champ nommé : dépendance explicite
	store  Store
	cfg    Config
}
```

Ici, `s.logger.Info(...)` dit d'où vient le comportement. Avec `*slog.Logger` embarqué,
`s.Info(...)` fonctionnerait — mais `Server` exposerait alors publiquement **toute** l'API du
logger, y compris des méthodes qui n'ont rien à faire dans son contrat.

**Règle pratique :** embarquer quand le type extérieur doit réellement **être** utilisable
comme le type intérieur (décorateur, extension d'un contrat). Utiliser un champ nommé dans
tous les autres cas — c'est-à-dire la plupart du temps.

### Un mot sur `sync.Mutex` embarqué

```go
type Counter struct {
	sync.Mutex        // embarqué : c.Lock() fonctionne
	n int
}
```

Idiome répandu, mais discutable : il **exporte** `Lock()` et `Unlock()` dans l'API publique
de `Counter`. N'importe qui peut alors verrouiller le mutex depuis l'extérieur et provoquer
un interblocage. La forme prudente est un champ nommé non exporté :

```go
type Counter struct {
	mu sync.Mutex
	n  int
}
```

On y reviendra au niveau 6. C'est un bon exemple de la question à se poser à chaque
embedding : **suis-je en train d'élargir mon API publique sans le vouloir ?**

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

// --- Composition d'interfaces ---

type Reader interface{ Read() (string, error) }
type Closer interface{ Close() error }

type ReadCloser interface { // composée, pas écrite d'un bloc
	Reader
	Closer
}

// --- Implémentation concrète ---

type StringSource struct {
	data   []string
	pos    int
	closed bool
}

func (s *StringSource) Read() (string, error) {
	if s.closed {
		return "", errors.New("source fermée")
	}
	if s.pos >= len(s.data) {
		return "", errors.New("fin de source")
	}
	s.pos++
	return s.data[s.pos-1], nil
}

func (s *StringSource) Close() error { s.closed = true; return nil }

// --- Décorateur par embedding d'interface ---

// UpperReader décore un ReadCloser : il redéfinit Read et laisse Close être
// PROMU depuis l'interface embarquée. Aucune délégation à écrire pour Close.
type UpperReader struct {
	ReadCloser
}

func (u UpperReader) Read() (string, error) {
	s, err := u.ReadCloser.Read() // délégation explicite au décoré
	if err != nil {
		return "", err
	}
	return strings.ToUpper(s), nil
}

// --- Embedding de struct ---

type Base struct{ Name string }

func (b Base) Describe() string { return "je m'appelle " + b.Greet() }
func (b Base) Greet() string    { return "Base:" + b.Name }

type Derived struct {
	Base
	Extra string
}

func (d Derived) Greet() string { return "Derived:" + d.Name + "/" + d.Extra }

func main() {
	src := &StringSource{data: []string{"alpha", "beta"}}

	var rc ReadCloser = UpperReader{ReadCloser: src}
	for {
		s, err := rc.Read()
		if err != nil {
			fmt.Println("arrêt :", err)
			break
		}
		fmt.Println(s) // ALPHA, BETA
	}
	_ = rc.Close() // méthode PROMUE depuis l'interface embarquée

	d := Derived{Base: Base{Name: "Rex"}, Extra: "chien"}
	fmt.Println(d.Greet())        // Derived:Rex/chien — la méthode extérieure masque
	fmt.Println(d.Base.Greet())   // Base:Rex          — l'originale reste accessible
	fmt.Println(d.Describe())     // « je m'appelle Base:Rex » ← PAS Derived !
	fmt.Println(d.Name)           // Rex — champ promu

	// var b Base = d            // ← ne compilerait pas : pas de sous-typage
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `ReadCloser` composée | Deux interfaces à une méthode, combinées. C'est le modèle de `io.ReadCloser`. |
| `UpperReader{ReadCloser}` | Interface embarquée : `Close()` est promue, on n'écrit que `Read()`. Sur une interface à dix méthodes, l'économie est décisive. |
| `u.ReadCloser.Read()` | Délégation **explicite** au décoré. Écrire `u.Read()` provoquerait une récursion infinie. |
| `d.Describe()` → `Base:Rex` | **Le point capital.** `Describe` appartient à `Base` et appelle `Base.Greet`, pas `Derived.Greet`. Aucun dispatch virtuel. |
| `d.Base.Greet()` | La méthode masquée reste accessible par le nom du champ embarqué. |
| `var b Base = d` commenté | Il n'y a pas de sous-typage : `Dog` n'est pas un `Animal`. |

## Erreurs fréquentes

1. **Prendre l'embedding pour de l'héritage** et attendre du polymorphisme : le code du type embarqué n'appellera jamais la méthode redéfinie.
2. **Oublier la délégation explicite** dans un décorateur → récursion infinie.
3. **Champ embarqué `nil`** (interface ou pointeur) : panique sur toute méthode non redéfinie.
4. **Ambiguïté de promotion** entre deux types embarqués : erreur de compilation, il faut désambiguïser.
5. **Embarquer pour économiser trois caractères** : `s.logger.Info()` est plus clair que `s.Info()`.
6. **Élargir son API publique sans le vouloir** : embarquer un type exporte toutes ses méthodes exportées.
7. **Embarquer `sync.Mutex` publiquement** : `Lock()` devient appelable de l'extérieur.
8. **Embedding en cascade sur trois niveaux** : la promotion devient impossible à suivre.
9. **Croire que la promotion copie** : `d.Name` *est* `d.Animal.Name`, pas une copie.

## Bonnes pratiques Go

- **Champ nommé par défaut**, embedding seulement quand le type extérieur doit se comporter
  comme le type intérieur.
- Composer de **petites interfaces** plutôt qu'en écrire de grosses.
- Dans un décorateur, toujours déléguer via le nom du champ embarqué.
- Documenter ce qu'un embedding ajoute à l'API publique du type.
- Ne pas dépasser un niveau d'embedding, sauf raison forte.
- Pour un comportement configurable, préférer une **fonction** ou une **interface** en champ
  plutôt qu'une hiérarchie de types embarqués.
- `sync.Mutex` en champ **non exporté et nommé**.

## Ce que je dois retenir

- Un champ **anonyme** est embarqué ; ses champs et méthodes sont **promus**.
- Ce n'est **pas de l'héritage** : ni sous-typage, ni dispatch virtuel.
- Le code du type embarqué appelle **toujours** ses propres méthodes.
- La promotion se résout par **profondeur** ; l'ambiguïté est une erreur de compilation.
- Les interfaces se **composent** ; c'est ainsi qu'est bâti `io`.
- Embarquer une **interface** dans une struct donne le décorateur et le faux partiel.
- Embarquer **élargit l'API publique** : le faire délibérément, jamais par confort.

➡️ [Exercices](exercices.md)
