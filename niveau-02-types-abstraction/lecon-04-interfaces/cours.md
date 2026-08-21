# Leçon 4 — Interfaces

> La leçon la plus « culturelle » de la formation : c'est ici que le style Go se distingue
> vraiment de Java, C# ou Python. Elle s'appuie directement sur l'**ensemble de méthodes**
> vu à la leçon 3 — si cette notion est floue, y revenir d'abord.

## Objectifs

1. Comprendre la **satisfaction implicite** et pourquoi elle change la conception.
2. Concevoir de **petites** interfaces, définies côté consommateur.
3. Maîtriser l'assertion de type et le `switch` de type.
4. Comprendre le piège de l'**interface nil contenant un pointeur nil**.

## Explication

### Une interface est un ensemble de méthodes

```go
type Shape interface {
	Area() float64
	Perimeter() float64
}

type Circle struct{ R float64 }

func (c Circle) Area() float64      { return math.Pi * c.R * c.R }
func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.R }
```

`Circle` satisfait `Shape`. **Nulle part il n'est écrit qu'elle l'implémente.** Il n'y a pas
d'`implements`, pas d'annotation, pas d'héritage. Il suffit que les méthodes existent.

Cette **satisfaction implicite** (*structural typing*) a une conséquence de conception
majeure : on peut créer une interface **après coup** pour un type qu'on ne contrôle pas.
Écrire une interface qui décrit ce dont *on* a besoin, et les types existants la satisfont
sans être modifiés. Impossible en Java sans adaptateur.

Pour vérifier à la compilation qu'un type satisfait bien une interface, l'idiome est :

```go
var _ Shape = (*Circle)(nil)   // erreur de compilation si Circle ne satisfait pas Shape
```

### Ce qu'une interface contient réellement

Une valeur d'interface est un **couple (type dynamique, valeur)** :

```go
var s Shape        // (nil, nil) → s == nil
s = Circle{R: 2}   // (Circle, {2})
```

Ce détail explique le piège le plus célèbre de Go, plus bas.

### Petites interfaces : la culture Go

La bibliothèque standard donne le ton :

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Stringer interface { String() string }
type error interface { Error() string }
```

**Une méthode.** Un proverbe Go, de Rob Pike : *« The bigger the interface, the weaker the
abstraction »* — plus une interface est grosse, plus l'abstraction est faible.

`io.Reader` est probablement l'abstraction la plus rentable jamais écrite : un fichier, une
connexion réseau, un buffer mémoire, un décompresseur, une chaîne — tous des `io.Reader`, et
toute fonction qui en accepte un fonctionne avec les cinq. C'est ce qu'on visait dans le
projet 1 en acceptant un `io.Reader` plutôt qu'un nom de fichier.

### Définir l'interface **côté consommateur**

Règle fondamentale, et contre-intuitive pour qui vient de Java :

```go
// MAUVAIS : le paquet qui fournit l'implémentation définit l'interface
package storage
type Store interface { Get(id string) (*User, error); Put(u *User) error; Delete(...) ... }
type PostgresStore struct{}

// BON : le paquet qui CONSOMME définit ce dont il a besoin
package service
type userGetter interface {          // non exportée, minuscule, une méthode
	Get(id string) (*User, error)
}
func NewService(g userGetter) *Service { … }
```

Pourquoi c'est mieux :

- L'interface décrit **le besoin réel**, pas la totalité de l'implémentation.
- Le consommateur peut la remplacer par un faux en test, trivialement.
- Le fournisseur ne dépend de rien ; il expose une struct concrète.
- Ajouter une méthode au fournisseur ne casse aucun consommateur.

Corollaire idiomatique : **« Accept interfaces, return structs »** — accepter des
interfaces en paramètre, retourner des types concrets. Retourner une interface prive
l'appelant des méthodes supplémentaires et rend le code plus difficile à faire évoluer.

Et surtout : **ne pas créer d'interface tant qu'il n'y a qu'une seule implémentation et
aucun besoin de test**. L'interface prématurée est la sur-ingénierie la plus fréquente en Go.

### `any` et l'assertion de type

`any` est un alias de `interface{}` (depuis Go 1.18) : l'interface vide, que **tout** type
satisfait.

```go
var x any = "bonjour"

s := x.(string)          // assertion : panique si x n'est pas un string
s, ok := x.(string)      // forme sûre : ok vaut false au lieu de paniquer
```

Toujours préférer la forme à deux résultats, sauf quand un échec serait un vrai bug.

### Le `switch` de type

```go
func describe(v any) string {
	switch val := v.(type) {
	case nil:
		return "nil"
	case int:
		return fmt.Sprintf("entier %d", val)      // val est un int ici
	case string:
		return fmt.Sprintf("chaîne %q", val)      // val est un string ici
	case []int:
		return fmt.Sprintf("slice de %d entiers", len(val))
	case error:
		return "erreur : " + val.Error()          // les interfaces sont acceptées
	default:
		return fmt.Sprintf("type inconnu %T", val)
	}
}
```

`val` prend le type du cas correspondant — c'est la seule construction du langage où une
variable change de type selon la branche. Attention : dans un `case` à plusieurs types,
`val` garde le type de l'interface d'origine.

**Un `switch` de type sur un type qu'on contrôle est presque toujours un signe qu'il fallait
une méthode.** Il est légitime pour du décodage (JSON, `any`), pour les erreurs, pour du
code générique — pas pour simuler du polymorphisme.

### Le piège : interface nil ≠ pointeur nil

```go
type MyErr struct{}
func (e *MyErr) Error() string { return "boom" }

func mayFail() error {
	var e *MyErr = nil    // pointeur nil
	return e              // ← converti en interface : (type=*MyErr, valeur=nil)
}

if err := mayFail(); err != nil {
	fmt.Println("erreur détectée !")   // ← S'AFFICHE ! Pourtant e était nil…
}
```

Pourquoi : l'interface contient le couple (`*MyErr`, `nil`). Le **type** n'est pas nil, donc
l'interface n'est pas nil. Une interface ne vaut `nil` que si son type **et** sa valeur le sont.

La parade est simple et absolue : **ne jamais déclarer une variable de type pointeur concret
pour la retourner en `error`**. Retourner `nil` littéralement :

```go
func mayFail() error {
	if ok {
		return nil        // ← le vrai nil
	}
	return &MyErr{}
}
```

Ce bug a mordu absolument tout le monde, y compris dans du code de production largement
diffusé. Le connaître est un marqueur de niveau.

### Ce qui vient ensuite : la composition

Une interface peut en **embarquer** d'autres, et une struct peut embarquer un type pour en
promouvoir les méthodes. Ces deux mécanismes forment la réponse de Go à l'héritage, et font
l'objet de la [leçon 5](../lecon-05-composition/cours.md).

Ce qu'il faut retenir dès maintenant : Go n'a **ni classes, ni héritage, ni polymorphisme
d'implémentation**. Il a les interfaces pour le polymorphisme — sans aucun lien de parenté
entre les types — l'embedding pour réutiliser du code, et les fonctions de première classe
pour le comportement paramétrable (niveau 1, leçon 11). Ces trois outils couvrent l'essentiel
de ce que fait l'héritage, sans hiérarchies fragiles.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"os"
	"sort"
	"strings"
)

// Notifier décrit ce dont le service a besoin : UNE méthode.
// Définie côté consommateur, non exportée si elle reste interne.
type Notifier interface {
	Notify(user, message string) error
}

// EmailNotifier et LogNotifier ne déclarent nulle part qu'elles implémentent Notifier.
type EmailNotifier struct{ From string }

func (e EmailNotifier) Notify(user, msg string) error {
	if !strings.Contains(user, "@") {
		return fmt.Errorf("adresse invalide : %q", user)
	}
	fmt.Printf("[mail de %s à %s] %s\n", e.From, user, msg)
	return nil
}

type LogNotifier struct{ Out *os.File }

func (l LogNotifier) Notify(user, msg string) error {
	_, err := fmt.Fprintf(l.Out, "[log] %s : %s\n", user, msg)
	return err
}

// Vérification à la compilation : l'idiome standard.
var (
	_ Notifier = EmailNotifier{}
	_ Notifier = LogNotifier{}
)

// Broadcast accepte une INTERFACE (petite) et retourne une erreur agrégée.
func Broadcast(n Notifier, users []string, msg string) error {
	var errs []error
	for _, u := range users {
		if err := n.Notify(u, msg); err != nil {
			errs = append(errs, fmt.Errorf("%s : %w", u, err))
		}
	}
	return errors.Join(errs...) // nil si errs est vide
}

// sortByLen montre une autre forme de polymorphisme : la fonction en paramètre.
func sortByLen(items []string) {
	sort.Slice(items, func(i, j int) bool { return len(items[i]) < len(items[j]) })
}

func main() {
	users := []string{"alice@x.fr", "bob-sans-arobase", "carol@y.fr"}

	notifiers := []Notifier{
		EmailNotifier{From: "no-reply@app.fr"},
		LogNotifier{Out: os.Stdout},
	}

	for _, n := range notifiers {
		fmt.Printf("--- %T ---\n", n)
		if err := Broadcast(n, users, "maintenance à 22h"); err != nil {
			fmt.Fprintf(os.Stderr, "échecs :\n%v\n", err)
		}
	}

	// Le piège de l'interface nil, démontré :
	var ln *LogNotifier
	var iface Notifier = ln
	fmt.Println("\nln == nil :", ln == nil)         // true
	fmt.Println("iface == nil :", iface == nil)     // FALSE — le type n'est pas nil
	fmt.Printf("iface = (%T, %v)\n", iface, iface)
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `type Notifier interface { Notify(…) error }` | Une seule méthode. Ajouter `Close()` ou `Configure()` la rendrait moins réutilisable. |
| Aucun `implements` | La satisfaction est implicite. Les deux types ne connaissent même pas l'existence de `Notifier`. |
| `var _ Notifier = EmailNotifier{}` | Vérification à la compilation, à coût nul à l'exécution. Idiome standard. |
| `Broadcast(n Notifier, …)` | Accepte une interface, ce qui la rend testable avec un faux en trois lignes. |
| `errors.Join(errs...)` | Go 1.20+ : agrège des erreurs, retourne `nil` si le slice est vide. Réponse à une question posée au niveau 1, leçon 10. |
| `%T` | Affiche le **type dynamique** — l'outil de diagnostic n°1 avec les interfaces. |
| La démonstration finale | `ln == nil` est vrai, `iface == nil` est faux. À exécuter soi-même pour que ça marque. |

## Erreurs fréquentes

1. **Créer une interface pour une seule implémentation, sans besoin de test.** Sur-ingénierie.
2. **Interfaces trop grosses** : cinq méthodes, personne ne peut en faire un faux simple.
3. **Définir l'interface côté fournisseur** au lieu du consommateur.
4. **Retourner une interface** au lieu d'un type concret (`return interfaces, accept structs` est l'inverse de la règle).
5. **Le piège nil** : retourner un pointeur concret nil comme `error`.
6. **Assertion sans le `, ok`** là où l'échec est possible → panique.
7. **`switch` de type sur des types qu'on contrôle** : il fallait une méthode.
8. **`any` partout** : on perd tout le bénéfice du typage statique. Depuis Go 1.18, les
   génériques (niveau 2) remplacent la plupart des usages d'`any`.
9. **Mélanger récepteurs valeur et pointeur** : seul `*T` satisfait alors l'interface, et le
   message d'erreur (« method has pointer receiver ») déroute. C'est la raison profonde de
   la règle de cohérence de la leçon 3.

## Bonnes pratiques Go

- **Petites interfaces**, une à trois méthodes.
- Les définir **là où on les consomme**, non exportées quand elles restent internes.
- *Accept interfaces, return structs.*
- Ne pas abstraire avant d'avoir **deux** implémentations réelles ou un besoin de test avéré.
- Nommer les interfaces à une méthode par l'agent : `Reader`, `Writer`, `Notifier`, `Formatter`.
- `var _ Iface = (*T)(nil)` pour verrouiller le contrat à la compilation.
- Ne jamais retourner un pointeur concret dans une variable de type `error`.
- Réutiliser les interfaces de la stdlib (`io.Reader`, `io.Writer`, `fmt.Stringer`, `error`)
  plutôt que d'en inventer d'équivalentes.

## Ce que je dois retenir

- La satisfaction d'interface est **implicite** : aucune déclaration nécessaire.
- Une interface est un couple **(type dynamique, valeur)** — d'où le piège du nil.
- **Une interface n'est `nil` que si type ET valeur sont nil.**
- *The bigger the interface, the weaker the abstraction.*
- Définir les interfaces **côté consommateur** ; accepter des interfaces, retourner des structs.
- `v, ok := x.(T)` plutôt que `x.(T)`.
- Le polymorphisme passe **uniquement** par les interfaces ; la réutilisation de code passe par la composition (leçon 5).
- Ne pas abstraire prématurément.

---

🧵 **Fil rouge :** cette leçon ajoute une ligne au tableau *[Copié ou partagé ?](../../ressources/copie-ou-partage.md)* — c'est le moment de le relire.

➡️ [Exercices](exercices.md)
