# Leçon 8 — Packages et visibilité

## Objectifs

1. Découper un programme en packages et comprendre ce qui les délimite.
2. Maîtriser la visibilité par la casse, et ce qu'elle protège réellement.
3. Éviter les cycles d'import — contrainte stricte de Go, et pourquoi elle est saine.
4. Connaître `internal/`, `init()`, les imports nommés et l'import anonyme.

## Explication

### Le package est le répertoire

Rappel du niveau 1 : tous les fichiers `.go` d'un même répertoire appartiennent au **même**
package, et ils se voient mutuellement **sans import**. Le découpage en fichiers est un
confort de lecture ; le découpage en **répertoires** est le vrai découpage architectural.

```
monprojet/
├── go.mod                  module github.com/user/monprojet
├── main.go                 package main
├── store/
│   ├── store.go            package store
│   └── memory.go           package store  ← même package que store.go
└── api/
    └── handler.go          package api
```

```go
import "github.com/user/monprojet/store"

s := store.New()
```

Le chemin d'import est **chemin du module + chemin relatif du répertoire**. Le nom utilisé
dans le code est le nom déclaré par `package`, pas le dernier segment du chemin — ils sont
identiques par convention, et s'en écarter est une source de confusion.

### La visibilité tient en une règle

**Un identifiant commençant par une majuscule est exporté ; sinon il est interne au package.**

```go
package store

type User struct {
	ID   int      // exporté
	Name string   // exporté
	salt []byte   // NON exporté : invisible hors du package
}

func New() *Store       { … }   // exporté
func validate(u User) error { … } // interne
```

Cela s'applique aux types, fonctions, méthodes, champs, constantes et variables. Il n'y a ni
`public`, ni `private`, ni `protected`, ni `friend`. Le **package** est l'unité
d'encapsulation : à l'intérieur, tout est accessible ; à l'extérieur, seuls les identifiants
exportés le sont.

Conséquence de conception : **l'encapsulation en Go se pense au niveau du package**, pas du
type. Deux types du même package peuvent tripatouiller mutuellement leurs champs privés —
c'est voulu, et c'est pourquoi un package doit rester cohérent.

### Nommer un package

Le nom du package **fait partie** du nom de tout ce qu'il exporte. `store.New()`,
`http.Client`, `bytes.Buffer` se lisent comme des phrases.

Règles :
- **Court, en minuscules, un seul mot**, sans souligné ni majuscule : `store`, `httputil`,
  `strconv`.
- **Pas de bégaiement** : `store.StoreUser` est mauvais, `store.User` est bon.
  `http.HTTPServer` serait absurde ; c'est `http.Server`.
- **Pas de fourre-tout** : `util`, `common`, `helpers`, `base`, `misc` ne disent rien de ce
  qu'ils contiennent et grossissent indéfiniment. Si un package s'appelle `util`, c'est qu'on
  n'a pas décidé de son rôle.
- Le nom décrit **ce que le package fournit**, pas ce qu'il contient : `list`, pas `lists` ;
  `sort`, pas `sorting`.

### Les cycles d'import sont interdits

```
package a  →  importe b
package b  →  importe a          ← ERREUR : import cycle not allowed
```

Go refuse **catégoriquement** les cycles d'import, à la compilation. Ce n'est pas une
limitation à contourner : c'est une contrainte qui force à réfléchir au sens des dépendances.

Trois façons de casser un cycle :

1. **Extraire ce qui est partagé** dans un troisième package dont les deux dépendent.
   C'est presque toujours la bonne réponse — le cycle révèle qu'un concept commun n'a pas été
   nommé.
2. **Inverser une dépendance avec une interface**, définie côté consommateur (leçon 4).
   `a` définit l'interface dont il a besoin ; `b` l'implémente sans le savoir et sans importer `a`.
3. **Fusionner** les deux packages, s'ils sont en réalité un seul concept découpé arbitrairement.

Un cycle n'est jamais un problème d'outillage : c'est un problème de conception rendu visible.

### `internal/` : la visibilité à l'échelle du projet

Un répertoire nommé `internal` crée une barrière :

```
github.com/user/monprojet/
├── internal/
│   └── auth/           importable UNIQUEMENT depuis github.com/user/monprojet/…
└── pkg/
    └── client/         importable par n'importe qui
```

Un package sous `internal/` n'est importable que par du code dont le chemin de module partage
le parent d'`internal`. Le compilateur le vérifie.

C'est le seul mécanisme de Go pour dire « exporté, mais pas pour vous ». Il est
**extrêmement utile** : tout ce qui n'est pas destiné à être une API publique stable devrait
y vivre. Beaucoup de projets Go professionnels mettent 90 % de leur code sous `internal/`.

### Imports : les formes à connaître

```go
import (
	"fmt"                                  // standard
	"os"

	"github.com/user/projet/store"         // interne au projet

	"github.com/lib/pq"                    // tiers
)

import mrand "math/rand"                   // import NOMMÉ : lève une collision
import _ "github.com/lib/pq"               // import ANONYME : pour ses effets de bord
import . "fmt"                             // import POINT : à PROSCRIRE
```

- **Import nommé** : indispensable quand deux packages portent le même nom
  (`math/rand` et `crypto/rand`), ou quand le nom est trop générique.
- **Import anonyme (`_`)** : le package est chargé pour ses `init()` uniquement. Cas typique :
  l'enregistrement d'un pilote SQL (`database/sql` niveau 8) ou d'un format d'image.
  **C'est un effet de bord invisible** : il mérite toujours un commentaire.
- **Import point (`.`)** : injecte les identifiants dans l'espace de noms courant. Rend le
  code illisible (« d'où vient cette fonction ? ») et casse les outils. À ne jamais utiliser
  hors de cas très particuliers en test.

`gofmt` regroupe et trie les imports ; `goimports` les ajoute et les retire automatiquement.
La convention est un groupe pour la bibliothèque standard, un pour le projet, un pour les
tiers — séparés par une ligne vide.

### `init()` : à utiliser avec méfiance

```go
func init() {
	// exécutée automatiquement avant main(), après l'initialisation des variables
}
```

Un package peut avoir plusieurs `init()`, dans plusieurs fichiers. Ils s'exécutent après
l'initialisation des variables de package, dans l'ordre des fichiers — c'est-à-dire un ordre
sur lequel il ne faut **pas** compter.

Problèmes : `init()` s'exécute même si la fonctionnalité n'est jamais utilisée, ne peut pas
retourner d'erreur (donc panique en cas de problème), et rend les tests difficiles car on ne
peut pas la désactiver.

**Préférer une initialisation explicite** : une fonction `New()` appelée depuis `main`, ou
`sync.OnceValue` (niveau 6) pour une initialisation paresseuse. `init()` se justifie
essentiellement pour l'**enregistrement dans un registre** — pilotes SQL, formats d'image,
handlers de sérialisation — c'est-à-dire le cas où l'import anonyme a du sens.

### Ce qui fait un bon découpage

Un package doit avoir une **raison d'exister formulable en une phrase**. Signaux d'alerte :

- il s'appelle `util`, `common` ou `models` ;
- il contient un seul type et trois lignes — le découpage est trop fin, cela crée des cycles
  et de la friction sans bénéfice ;
- il contient quarante fichiers sans rapport — trop grossier ;
- on n'arrive pas à décider dans quel package mettre une nouvelle fonction.

Le découpage **par couche technique** (`models`, `services`, `handlers`) est le réflexe des
développeurs venus de Java ou de Rails. Il fonctionne sur un petit projet, mais tend à créer
des packages sans cohésion et des cycles. Le découpage **par domaine** (`user`, `billing`,
`inventory`, chacun contenant son modèle, sa logique et son stockage) résiste mieux à la
croissance. C'est le sujet de la leçon 10 et du niveau 10.

## Exemple

Arborescence :

```
biblio/
├── go.mod                       module exemple.com/biblio
├── main.go                      package main
├── internal/
│   └── catalog/
│       ├── catalog.go           package catalog
│       └── isbn.go              package catalog
└── format/
    └── format.go                package format
```

```go
// internal/catalog/catalog.go
// Package catalog gère le référentiel de livres.
// Il est sous internal/ : aucune application extérieure ne peut en dépendre,
// ce qui laisse toute liberté pour le faire évoluer.
package catalog

import (
	"errors"
	"fmt"
	"maps"
	"slices"
)

var ErrNotFound = errors.New("livre introuvable")

// Book est exporté : c'est le vocabulaire du package.
type Book struct {
	ISBN   string
	Title  string
	Author string
	copies int // NON exporté : l'invariant « jamais négatif » est protégé
}

func (b Book) Copies() int { return b.copies }

// Catalog est le type principal. Le nom du package évite le bégaiement :
// on écrira catalog.Catalog… ce qui bégaie justement. Voir la remarque plus bas.
type Catalog struct {
	books map[string]*Book
}

func New() *Catalog { return &Catalog{books: make(map[string]*Book)} }

func (c *Catalog) Add(b Book, copies int) error {
	if err := validISBN(b.ISBN); err != nil { // fonction interne, non exportée
		return fmt.Errorf("ajout de %q : %w", b.Title, err)
	}
	if copies < 0 {
		return fmt.Errorf("nombre d'exemplaires négatif : %d", copies)
	}
	b.copies = copies
	c.books[b.ISBN] = &b
	return nil
}

func (c *Catalog) Borrow(isbn string) error {
	b, ok := c.books[isbn]
	if !ok {
		return fmt.Errorf("emprunt de %s : %w", isbn, ErrNotFound)
	}
	if b.copies == 0 {
		return fmt.Errorf("emprunt de %s : aucun exemplaire disponible", isbn)
	}
	b.copies--
	return nil
}

// All retourne les livres triés par ISBN : ordre déterministe.
func (c *Catalog) All() []Book {
	out := make([]Book, 0, len(c.books))
	for _, k := range slices.Sorted(maps.Keys(c.books)) {
		out = append(out, *c.books[k]) // copie : l'appelant ne peut pas casser l'invariant
	}
	return out
}
```

```go
// internal/catalog/isbn.go — MÊME package, autre fichier.
package catalog

import (
	"errors"
	"strings"
)

// validISBN n'est pas exportée : elle appartient à l'implémentation.
// Elle est visible depuis catalog.go sans aucun import.
func validISBN(s string) error {
	s = strings.ReplaceAll(s, "-", "")
	if len(s) != 13 {
		return errors.New("un ISBN doit comporter 13 chiffres")
	}
	for _, r := range s {
		if r < '0' || r > '9' {
			return errors.New("un ISBN ne contient que des chiffres")
		}
	}
	return nil
}
```

```go
// format/format.go — package indépendant, SANS dépendance vers catalog.
// C'est ce qui évite le cycle : format ne connaît que ce dont il a besoin.
package format

import (
	"fmt"
	"strings"
)

// Row est l'interface définie CÔTÉ CONSOMMATEUR (leçon 4) : format décrit
// ce dont il a besoin, pas ce que catalog fournit.
type Row interface {
	Columns() []string
}

func Table(header []string, rows []Row) string {
	var b strings.Builder
	fmt.Fprintln(&b, strings.Join(header, " | "))
	for _, r := range rows {
		fmt.Fprintln(&b, strings.Join(r.Columns(), " | "))
	}
	return b.String()
}
```

```go
// main.go
package main

import (
	"fmt"
	"os"

	"exemple.com/biblio/format"
	"exemple.com/biblio/internal/catalog"
)

// bookRow adapte catalog.Book à format.Row.
// L'adaptation se fait ICI, dans main, qui connaît les deux — donc ni catalog
// ni format ne dépendent l'un de l'autre. Aucun cycle possible.
type bookRow struct{ b catalog.Book }

func (r bookRow) Columns() []string {
	return []string{r.b.ISBN, r.b.Title, r.b.Author, fmt.Sprint(r.b.Copies())}
}

func main() {
	c := catalog.New()
	if err := c.Add(catalog.Book{ISBN: "9781234567897", Title: "Le Go", Author: "A. B."}, 3); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if err := c.Add(catalog.Book{ISBN: "abc", Title: "Invalide"}, 1); err != nil {
		fmt.Fprintln(os.Stderr, "refusé :", err)
	}

	rows := make([]format.Row, 0)
	for _, b := range c.All() {
		rows = append(rows, bookRow{b})
	}
	fmt.Print(format.Table([]string{"ISBN", "Titre", "Auteur", "Ex."}, rows))

	// b.copies = 99   ← impossible depuis main : champ non exporté
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `internal/catalog` | Aucun projet extérieur ne peut l'importer : liberté totale d'évolution. |
| `validISBN` dans un autre fichier | Même package, donc visible sans import. Le découpage en fichiers est purement organisationnel. |
| `copies` non exporté + `Copies()` | L'invariant est protégé **par le package**, pas par le type. |
| `All()` retourne des copies | L'appelant ne peut pas modifier l'état interne. |
| `format.Row` définie côté consommateur | `format` ne dépend pas de `catalog`. Sans cela, un cycle apparaîtrait dès que `catalog` voudrait s'afficher. |
| `bookRow` dans `main` | L'adaptation vit là où les deux mondes se rencontrent — le point le plus haut. C'est le patron d'adaptateur. |
| `catalog.Catalog` | **Bégaiement assumé, et c'est un défaut.** Le package aurait dû s'appeler `books` avec un type `Catalog`, ou le type s'appeler simplement `catalog.New()` retournant un type non exporté derrière une interface. À noter comme un vrai point de revue de code. |

## Erreurs fréquentes

1. **Package `util` / `common` / `helpers`** : fourre-tout qui grossit sans fin.
2. **Bégaiement** : `store.StoreClient`, `user.UserService`.
3. **Créer un cycle** puis chercher à le contourner au lieu de corriger la conception.
4. **Tout exporter par réflexe** : plus rien n'est protégeable, et chaque champ devient une API publique à maintenir.
5. **Ne rien mettre sous `internal/`** : tout le projet devient une API publique involontaire.
6. **Découper trop tôt** : dix packages de trente lignes créent plus de friction que de clarté.
7. **Logique dans `init()`** : difficile à tester, impossible à désactiver, ne peut pas retourner d'erreur.
8. **Import point** : on ne sait plus d'où viennent les identifiants.
9. **Import anonyme non commenté** : le lecteur ne comprend pas pourquoi il est là.
10. **Nom de package différent du nom du répertoire** : source de confusion permanente.

## Bonnes pratiques Go

- Un package = une raison d'exister, formulable en une phrase.
- Nom court, singulier, minuscule, sans souligné. Le nom du package fait partie du nom des identifiants.
- **Tout sous `internal/`** sauf ce qui est délibérément une API publique.
- Exporter le minimum ; il est facile d'exporter plus tard, impossible de retirer.
- Casser les cycles en extrayant un package commun ou en inversant avec une interface.
- Commentaire de package (`// Package x …`) dans un seul fichier, souvent `doc.go`.
- Éviter `init()` ; préférer une initialisation explicite.
- Découper quand la douleur apparaît, pas par anticipation.

## Ce que je dois retenir

- **Un package = un répertoire** ; les fichiers d'un même répertoire se voient sans import.
- **La casse de la première lettre** est le seul mécanisme de visibilité.
- L'encapsulation se pense **au niveau du package**, pas du type.
- Les **cycles d'import sont interdits** : un cycle révèle un défaut de conception.
- **`internal/`** restreint l'import au projet lui-même : à utiliser massivement.
- `init()` et l'import anonyme sont des mécanismes à effet de bord, à réserver aux registres.
- Pas de `util`, pas de bégaiement, pas de découpage prématuré.

➡️ [Exercices](exercices.md)
