# Leçon 4 — Interfaces bien conçues

> Les interfaces sont l'outil d'architecture principal de Go : elles découplent un consommateur de ses
> dépendances. Mal utilisées, elles produisent l'inverse — une interface par struct, déclarée « au cas
> où », de quinze méthodes, que chaque test doit simuler en entier. La règle Go tient en une phrase,
> répétée depuis N2 L4 : **l'interface appartient à celui qui l'utilise, et elle est petite**. Cette leçon
> en tire toutes les conséquences pour l'architecture.

## Objectifs

1. Déclarer les interfaces du côté du **consommateur**, avec les seules méthodes qu'il appelle.
2. Appliquer « accepter des interfaces, rendre des structs ».
3. Découvrir les interfaces quand le besoin apparaît, au lieu de les prévoir.
4. Écrire des faux minimaux, dont l'adaptateur fonction (`HandlerFunc`).
5. Reconnaître la pollution par les interfaces et le piège du faux partiel par embarquement.

## Explication

### Côté consommateur, et petite

Dans d'autres langages, une classe **déclare** les interfaces qu'elle implémente : l'interface est
écrite par le fournisseur, souvent avec toutes ses méthodes. En Go, l'implémentation est **implicite** :
un type satisfait toute interface dont il a les méthodes, sans le savoir. Conséquence : c'est le
**consommateur** qui déclare ce dont **il** a besoin.

```go
// à côté de RemindOverdue, et non à côté de Store :
type overdueLister interface {
	Overdue(ctx context.Context, now time.Time) ([]Loan, error)
}
```

Bénéfices :

- l'interface dit **exactement** ce que la fonction utilise : sa signature documente sa dépendance ;
- un faux de test implémente **une** méthode, pas quinze ;
- le fournisseur (`Store`) peut gagner des méthodes sans casser personne ;
- le consommateur n'importe pas le paquet du fournisseur pour l'interface : pas de dépendance vers la
  technologie (leçon 1).

La bibliothèque standard en est l'illustration : `io.Reader` (une méthode), `io.Writer` (une),
`fmt.Stringer` (une), `sort.Interface` (trois), `http.Handler` (une). Les interfaces plus grandes s'y
**composent** à partir des petites (`io.ReadWriteCloser`).

### Accepter des interfaces, rendre des structs

- **En paramètre**, une interface : la fonction accepte toute implémentation (le vrai stockage, un faux,
  un cache devant le stockage).
- **En retour**, une struct concrète : `NewStore` rend `*Store`. L'appelant dispose de **toutes** les
  méthodes, et décide lui-même de la petite interface par laquelle il la passera ailleurs. Rendre une
  interface (`func NewStore() StoreInterface`) cache des méthodes, empêche d'en ajouter sans casser les
  implémentations tierces, et force un `StoreInterface` côté fournisseur.

Exceptions légitimes au retour d'interface : `error` ; une fabrique qui choisit **réellement** entre
plusieurs implémentations (`crypto/sha256.New` rend un `hash.Hash`) ; masquer volontairement une
implémentation non exportée.

### Découvrir, ne pas prévoir

On n'écrit pas d'interface pour un type qui n'a qu'une implémentation **et** aucun besoin de
substitution. On l'extrait au moment où le besoin apparaît : un test qui veut un faux, une deuxième
implémentation, un découplage entre paquets. En Go, extraire une interface après coup est trivial (le type
existant la satisfait déjà, implicitement) : il n'y a aucune raison de la prévoir.

Signes de **pollution par les interfaces** :

- une interface par struct, avec les mêmes méthodes, souvent nommée `IStore` ou `StoreInterface` ;
- des interfaces déclarées dans le paquet du fournisseur « pour que les autres puissent mocker » ;
- des interfaces de dix méthodes dont chaque consommateur n'utilise que deux ;
- des faux de test générés de 300 lignes pour simuler ces interfaces.

### Des faux minimaux

Pour une interface d'une méthode, l'**adaptateur fonction** donne un faux en une ligne, comme
`http.HandlerFunc` pour `http.Handler` :

```go
type overdueFunc func(context.Context, time.Time) ([]Loan, error)

func (f overdueFunc) Overdue(ctx context.Context, now time.Time) ([]Loan, error) { return f(ctx, now) }
```

Pour plusieurs méthodes, une petite struct avec des champs (N5 L3).

### Le piège du faux partiel

Avec une grosse interface, il est tentant d'**embarquer** l'interface dans le faux et de ne redéfinir que
la méthode utile :

```go
type partialFake struct{ FatStore } // l'interface embarquée vaut nil
```

Ça compile, et ça passe… jusqu'au jour où le code testé appelle une autre méthode : déréférencement de
pointeur nil, **panique** en plein test, loin de la cause (mesuré dans l'exemple). C'est le symptôme d'une
interface trop grosse : la vraie correction est de la réduire, pas de la simuler partiellement.

### Vérifier à la compilation

```go
var _ overdueLister = (*Store)(nil)
```

Cette ligne ne fait rien à l'exécution, mais échoue à la **compilation** si `*Store` cesse de satisfaire
l'interface (méthode renommée, signature changée). Utile quand la conformité n'est vérifiée nulle part
ailleurs (un type utilisé seulement par réflexion, ou par un autre module).

### Quand ne pas le faire

- **Pas d'interface du tout** quand il n'y a qu'une implémentation, aucun test qui veuille la remplacer
  et aucune frontière de paquet à découpler : appeler la struct.
- **Pas une interface par consommateur** si dix fonctions d'un même paquet ont besoin des mêmes trois
  méthodes : une interface partagée **dans ce paquet** suffit.
- **Pas d'interface pour une fonction pure** ou un calcul : on ne le remplace jamais en test.

## Exemple

Un stockage concret, deux consommateurs qui déclarent chacun leur petite interface, un faux en une ligne,
et le contre-exemple du faux partiel.

```go
package main

import (
	"context"
	"fmt"
	"slices"
	"strings"
	"time"
)

type Loan struct {
	Member, Book string
	Due          time.Time
}

// --- Le fournisseur : une struct CONCRÈTE, avec toutes ses méthodes. Pas d'interface ici. ---

type Store struct{ loans []Loan }

func NewStore(loans ...Loan) *Store { return &Store{loans: loans} } // rend une struct

func (s *Store) Overdue(_ context.Context, now time.Time) ([]Loan, error) {
	var out []Loan
	for _, l := range s.loans {
		if now.After(l.Due) {
			out = append(out, l)
		}
	}
	return out, nil
}

func (s *Store) CountByMember(_ context.Context) (map[string]int, error) {
	m := map[string]int{}
	for _, l := range s.loans {
		m[l.Member]++
	}
	return m, nil
}

func (s *Store) Add(_ context.Context, l Loan) error { s.loans = append(s.loans, l); return nil }

// … imaginer ici une dizaine d'autres méthodes (Remove, Renew, History, Search…).

// --- Les consommateurs : chacun déclare la PETITE interface dont il a besoin. ---

// overdueLister : une méthode. Déclarée à côté de son unique utilisateur, non exportée.
type overdueLister interface {
	Overdue(ctx context.Context, now time.Time) ([]Loan, error)
}

// RemindOverdue n'a besoin que de lister les retards, et d'envoyer un message.
func RemindOverdue(ctx context.Context, src overdueLister, now time.Time, send func(to, msg string)) error {
	loans, err := src.Overdue(ctx, now)
	if err != nil {
		return err
	}
	for _, l := range loans {
		days := int(now.Sub(l.Due).Hours() / 24)
		send(l.Member, fmt.Sprintf("« %s » a %d jour(s) de retard", l.Book, days))
	}
	return nil
}

type loanCounter interface {
	CountByMember(ctx context.Context) (map[string]int, error)
}

func Report(ctx context.Context, src loanCounter) (string, error) {
	counts, err := src.CountByMember(ctx)
	if err != nil {
		return "", err
	}
	members := make([]string, 0, len(counts))
	for m := range counts {
		members = append(members, m)
	}
	slices.Sort(members)
	parts := make([]string, len(members))
	for i, m := range members {
		parts[i] = fmt.Sprintf("%s=%d", m, counts[m])
	}
	return strings.Join(parts, " "), nil
}

// Vérification à la compilation : *Store satisfait bien les deux interfaces.
var (
	_ overdueLister = (*Store)(nil)
	_ loanCounter   = (*Store)(nil)
)

// Un faux en une ligne : un type fonction qui satisfait l'interface (comme http.HandlerFunc).
type overdueFunc func(context.Context, time.Time) ([]Loan, error)

func (f overdueFunc) Overdue(ctx context.Context, now time.Time) ([]Loan, error) { return f(ctx, now) }

// --- Le contre-exemple : une grosse interface, et le faux « partiel » par embarquement. ---

type FatStore interface {
	Overdue(ctx context.Context, now time.Time) ([]Loan, error)
	CountByMember(ctx context.Context) (map[string]int, error)
	Add(ctx context.Context, l Loan) error
	// … et dix autres
}

type partialFake struct {
	FatStore // embarqué, NIL : seules les méthodes redéfinies existent vraiment
}

func (partialFake) Overdue(context.Context, time.Time) ([]Loan, error) { return nil, nil }

func main() {
	ctx := context.Background()
	day := func(d int) time.Time { return time.Date(2026, 3, d, 12, 0, 0, 0, time.UTC) }
	store := NewStore(
		Loan{"ada", "La Peste", day(1)},
		Loan{"bob", "Vol de nuit", day(10)},
		Loan{"ada", "L'Étranger", day(20)},
	)
	now := day(12)
	send := func(to, msg string) { fmt.Printf("  à %s : %s\n", to, msg) }

	fmt.Println("rappels (vrai stockage) :")
	RemindOverdue(ctx, store, now, send)

	fmt.Println("rappels (faux en une ligne) :")
	fake := overdueFunc(func(context.Context, time.Time) ([]Loan, error) {
		return []Loan{{"zoé", "Test", day(11)}}, nil
	})
	RemindOverdue(ctx, fake, now, send)

	r, _ := Report(ctx, store)
	fmt.Println("rapport :", r)

	// Le faux partiel compile, et passe tant qu'on n'appelle que ce qu'il redéfinit…
	var fat FatStore = partialFake{}
	_, err := fat.Overdue(ctx, now)
	fmt.Println("faux partiel, Overdue :", err)
	func() {
		defer func() { fmt.Println("faux partiel, CountByMember : panique :", recover()) }()
		fat.CountByMember(ctx)
	}()
}
```

Sortie :

```
rappels (vrai stockage) :
  à ada : « La Peste » a 11 jour(s) de retard
  à bob : « Vol de nuit » a 2 jour(s) de retard
rappels (faux en une ligne) :
  à zoé : « Test » a 1 jour(s) de retard
rapport : ada=2 bob=1
faux partiel, Overdue : <nil>
faux partiel, CountByMember : panique : runtime error: invalid memory address or nil pointer dereference
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `NewStore` rend `*Store` | Le fournisseur ne déclare aucune interface ; il rend une struct. |
| `overdueLister`, `loanCounter` | Une méthode chacune, déclarées à côté de leur utilisateur, non exportées. |
| `RemindOverdue(ctx, src overdueLister, …)` | Accepte le vrai stockage **et** le faux, sans le savoir. |
| `send func(to, msg string)` | Une dépendance d'une seule opération : une fonction suffit, même pas besoin d'interface. |
| `var _ overdueLister = (*Store)(nil)` | Vérification de conformité à la compilation. |
| `overdueFunc` | L'adaptateur fonction : un faux en une ligne. |
| `FatStore` + `partialFake` | La grosse interface et son faux partiel : `Overdue` marche, `CountByMember` panique. |

## Erreurs fréquentes

1. **Déclarer l'interface chez le fournisseur**, avec toutes ses méthodes.
2. **Une interface par struct**, nommée `IStore` ou `StoreInterface`.
3. **Rendre une interface** depuis un constructeur sans raison.
4. **Prévoir des interfaces** pour des types sans seconde implémentation ni besoin de test.
5. **Des faux partiels par embarquement** : paniques en test loin de la cause.
6. **Des interfaces de dix méthodes** dont chaque consommateur utilise deux.
7. **Oublier qu'une fonction suffit** pour une dépendance d'une seule opération.

## Bonnes pratiques Go

- L'interface est déclarée par le consommateur, avec les méthodes qu'il appelle.
- Une à trois méthodes ; composer les interfaces plus grandes à partir des petites.
- Paramètres : interfaces ; retours : structs (sauf `error` et vraies fabriques).
- Extraire une interface au moment où un besoin réel apparaît.
- Adaptateur fonction pour les interfaces d'une méthode ; petites structs sinon.
- `var _ I = (*T)(nil)` quand la conformité n'est vérifiée nulle part ailleurs.

## Ce que je dois retenir

- L'implémentation est implicite : c'est le consommateur qui déclare l'interface.
- Petite = facile à implémenter, à simuler, à comprendre.
- Accepter des interfaces, rendre des structs.
- Découvrir les interfaces, ne pas les prévoir.
- Le faux partiel par embarquement est le symptôme d'une interface trop grosse.

---

➡️ [Exercices](exercices.md)
