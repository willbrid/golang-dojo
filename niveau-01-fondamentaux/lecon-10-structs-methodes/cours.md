# Leçon 10 — Structs et méthodes

## Objectifs

1. Modéliser des données avec des structs et connaître les quatre façons de les instancier.
2. Choisir entre **récepteur valeur** et **récepteur pointeur** — la décision la plus
   structurante de cette leçon.
3. Savoir quand une struct est comparable avec `==`.
4. Implémenter `String()` et comprendre pourquoi `fmt` l'appelle automatiquement.

## Explication

### Déclarer et instancier

```go
type User struct {
	ID      int
	Name    string
	Email   string
	private bool     // non exporté : invisible hors du package
}
```

Quatre façons de créer une valeur :

```go
u1 := User{}                                    // zéro-valeur : tous les champs à zéro
u2 := User{ID: 1, Name: "Alice"}                // littéral nommé — LA forme à utiliser
u3 := User{1, "Alice", "a@x.fr", false}         // positionnel — À ÉVITER
u4 := &User{ID: 1}                              // pointeur vers une struct
var u5 User                                     // zéro-valeur, comme u1
```

Le littéral **positionnel** est fragile : ajouter un champ au type casse tous les appels,
et l'ordre est invisible à la lecture. Il n'est acceptable que pour de très petites structs
stables (`image.Point{1, 2}`). Le littéral **nommé** est la règle.

Les champs omis prennent leur zéro-valeur. On retrouve le principe de la leçon 2 :
concevoir ses structs pour que `var x T` soit directement utilisable.

### Champs et anonymat

```go
u.Name = "Bob"
fmt.Println(u.Email)

p := &u
p.Name = "Carol"    // équivalent à (*p).Name — Go déréférence automatiquement
```

Go déréférence les pointeurs automatiquement pour l'accès aux champs : `p.Name` fonctionne
sans écrire `(*p).Name`. Sucre syntaxique appréciable.

Les structs anonymes existent, surtout pour des tables de tests (niveau 3) :

```go
tests := []struct {
	name string
	in   int
	want int
}{
	{"zéro", 0, 0},
	{"positif", 2, 4},
}
```

### Comparabilité

```go
type Point struct{ X, Y int }
a, b := Point{1, 2}, Point{1, 2}
fmt.Println(a == b)    // true — comparaison champ par champ

type Bad struct{ Tags []string }
c, d := Bad{}, Bad{}
fmt.Println(c == d)    // ERREUR de compilation : struct contenant un slice
```

Une struct est comparable si **tous** ses champs le sont. Slices, maps et fonctions ne le
sont pas et contaminent la struct entière. Conséquence directe (leçon 7) : seule une struct
comparable peut servir de clé de map.

Pour comparer des structs contenant des slices : `reflect.DeepEqual` (lent, à réserver aux
tests) ou une méthode `Equal` écrite à la main (rapide et explicite).

### Les méthodes : une fonction avec un récepteur

```go
func (u User) FullName() string {          // récepteur VALEUR
	return u.Name + " <" + u.Email + ">"
}

func (u *User) SetName(name string) {      // récepteur POINTEUR
	u.Name = name
}
```

Une méthode est une fonction ordinaire dont le premier paramètre est écrit avant le nom.
Point important : **on ne peut définir des méthodes que sur des types déclarés dans le même
paquet**. Impossible d'ajouter une méthode à `string` ou à `time.Time` — mais on peut
déclarer `type MyString string` et lui en donner. Cette restriction évite les conflits
qu'on trouve dans les langages à *monkey patching*.

### Valeur ou pointeur ? La décision centrale

```go
func (u User) Rename(n string)  { u.Name = n }   // modifie une COPIE : sans effet
func (u *User) Rename(n string) { u.Name = n }   // modifie l'original
```

**Utiliser un récepteur pointeur si :**
1. la méthode **modifie** le récepteur ;
2. la struct est **grosse** (la copie coûterait) ;
3. le type contient un `sync.Mutex` ou un autre élément non copiable ;
4. **la cohérence l'exige** — voir plus bas.

**Utiliser un récepteur valeur si :**
1. le type est petit et immuable par nature (`time.Time`, `Point`) ;
2. la méthode ne modifie rien et le type est un scalaire ou une petite struct.

**La règle de cohérence, la plus importante :** si **une seule** méthode d'un type a un
récepteur pointeur, **toutes** devraient en avoir un. Mélanger les deux crée des surprises
avec les interfaces (leçon 11) et déroute le lecteur. En cas de doute : **récepteur
pointeur partout**. C'est le conseil du wiki officiel Go.

### L'appel automatique — et sa limite

Go convertit automatiquement entre valeur et pointeur lors des appels de méthode :

```go
u := User{}
u.SetName("x")     // Go réécrit (&u).SetName("x")

p := &User{}
fmt.Println(p.FullName())  // Go réécrit (*p).FullName()
```

Cela ne fonctionne que si la valeur est **adressable**. D'où :

```go
users := map[string]User{"a": {}}
users["a"].SetName("x")     // ERREUR : les entrées de map ne sont pas adressables
```

Il faut soit une `map[string]*User`, soit lire/modifier/réécrire. C'est le retour du point
vu en leçon 7, et une source classique de frustration.

### `String()` : l'interface `Stringer`

```go
func (u User) String() string {
	return fmt.Sprintf("User(%d, %s)", u.ID, u.Name)
}

fmt.Println(u)              // User(1, Alice) — appelé automatiquement
fmt.Printf("%v\n", u)       // idem
fmt.Printf("%s\n", u)       // idem
```

`fmt` teste à l'exécution si la valeur satisfait `fmt.Stringer` (`String() string`) et
l'appelle. C'est un premier contact avec les interfaces : aucune déclaration n'est
nécessaire, la méthode suffit.

**Piège mortel :** ne jamais utiliser `%v` sur le récepteur *dans* `String()` — cela
provoque une récursion infinie et un dépassement de pile. Utiliser les champs directement,
ou convertir vers le type sous-jacent.

Le récepteur choisi pour `String()` détermine ce qui l'active. Avec `func (u *User) String()`,
`fmt.Println(u)` sur une **valeur** n'appellera pas la méthode. C'est un piège fréquent, et
c'est le sujet réel de la leçon 11.

### Constructeurs

Go n'a pas de constructeurs. La convention est une fonction `NewXxx` :

```go
func NewUser(name, email string) (*User, error) {
	if name == "" {
		return nil, errors.New("nom vide")
	}
	return &User{ID: nextID(), Name: name, Email: email}, nil
}
```

Elle n'est nécessaire que si la construction demande une validation, une initialisation
(map, channel) ou un identifiant. **Si la zéro-valeur suffit, ne pas écrire de constructeur** :
`var b strings.Builder` fonctionne sans `NewBuilder()`, et c'est un critère de qualité d'API.

### Embedding : la composition avant l'héritage

```go
type Animal struct{ Name string }
func (a Animal) Speak() string { return a.Name + " fait un bruit" }

type Dog struct {
	Animal        // champ ANONYME : embedding
	Breed  string
}

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "berger"}
fmt.Println(d.Name)      // "Rex"  — promotion du champ
fmt.Println(d.Speak())   // méthode promue elle aussi
```

Les champs et méthodes du type embarqué sont **promus** : accessibles directement sur le
type extérieur. Ce n'est **pas de l'héritage** :

- pas de polymorphisme : un `Dog` n'est pas un `Animal`, on ne peut pas l'affecter à une
  variable de type `Animal` ;
- pas de surcharge virtuelle : si `Dog` définit son propre `Speak()`, il **masque** celui
  d'`Animal` mais le code d'`Animal` continue d'appeler la version d'`Animal`.

Ce dernier point élimine la fameuse « erreur du constructeur qui appelle une méthode
surchargée » des langages à héritage. La composition est plus prévisible ; c'est le
fondement de la leçon 11.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

// Account représente un compte bancaire. Le solde est en CENTIMES :
// jamais de flottant pour de l'argent (leçon 2).
type Account struct {
	ID      string
	Owner   string
	balance int64 // non exporté : l'invariant « jamais négatif » est protégé
}

var ErrInsufficientFunds = errors.New("fonds insuffisants")

// NewAccount valide et construit un compte. Un constructeur est justifié ici :
// il y a une validation à faire.
func NewAccount(id, owner string, initial int64) (*Account, error) {
	if strings.TrimSpace(id) == "" {
		return nil, errors.New("identifiant vide")
	}
	if initial < 0 {
		return nil, fmt.Errorf("solde initial négatif : %d", initial)
	}
	return &Account{ID: id, Owner: owner, balance: initial}, nil
}

// Balance ne modifie rien, mais garde un récepteur pointeur par COHÉRENCE
// avec Deposit et Withdraw.
func (a *Account) Balance() int64 { return a.balance }

func (a *Account) Deposit(amount int64) error {
	if amount <= 0 {
		return fmt.Errorf("montant invalide : %d", amount)
	}
	a.balance += amount
	return nil
}

func (a *Account) Withdraw(amount int64) error {
	if amount <= 0 {
		return fmt.Errorf("montant invalide : %d", amount)
	}
	if amount > a.balance {
		return fmt.Errorf("retrait de %d sur %d : %w", amount, a.balance, ErrInsufficientFunds)
	}
	a.balance -= amount
	return nil
}

// String satisfait fmt.Stringer. Attention : on n'utilise PAS %v sur a.
func (a *Account) String() string {
	return fmt.Sprintf("%s (%s) : %d,%02d €", a.ID, a.Owner, a.balance/100, a.balance%100)
}

func main() {
	acc, err := NewAccount("FR001", "Alice", 10_000)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(acc) // String() est appelée : acc est un *Account

	if err := acc.Withdraw(15_000); err != nil {
		fmt.Println("refusé :", err)
		fmt.Println("cause connue ?", errors.Is(err, ErrInsufficientFunds))
	}

	_ = acc.Deposit(2_500)
	fmt.Println(acc)
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `balance int64` non exporté | L'invariant « le solde ne peut pas devenir négatif » est garanti parce que **personne** hors du paquet ne peut écrire ce champ. C'est l'encapsulation en Go : par la casse, pas par des mots-clés. |
| Centimes en `int64` | Rappel de la leçon 2 : jamais de flottant pour de l'argent. |
| `func (a *Account) Balance()` | Ne modifie rien, mais **cohérence** : tous les récepteurs sont des pointeurs. |
| `%w` sur `ErrInsufficientFunds` | L'appelant peut tester la cause avec `errors.Is` tout en lisant un message riche. |
| `String()` sur `*Account` | `fmt.Println(acc)` fonctionne car `acc` est déjà un `*Account`. Sur une valeur `Account`, la méthode ne serait **pas** appelée — piège développé en leçon 11. |
| `10_000` | Séparateur de milliers, autorisé dans les littéraux numériques Go. |

## Erreurs fréquentes

1. **Récepteur valeur alors qu'on veut modifier** : la méthode s'exécute sans effet, sans erreur. Bug silencieux le plus fréquent de cette leçon.
2. **Mélanger récepteurs valeur et pointeur** sur un même type.
3. **Littéral positionnel** : casse dès qu'un champ est ajouté.
4. **`%v` sur le récepteur dans `String()`** → récursion infinie, `stack overflow`.
5. **Appeler une méthode à récepteur pointeur sur une entrée de map** → ne compile pas.
6. **Comparer avec `==` une struct contenant un slice** → ne compile pas.
7. **Écrire un constructeur inutile** quand la zéro-valeur suffit.
8. **Prendre l'embedding pour de l'héritage** : attendre du polymorphisme là où il n'y en a pas.
9. **Exporter tous les champs par réflexe** : plus aucun invariant n'est protégeable.

## Bonnes pratiques Go

- Littéraux **nommés**, toujours.
- **Cohérence des récepteurs** ; en cas de doute, pointeur partout.
- Champs non exportés dès qu'un invariant doit être protégé ; accesseurs seulement si nécessaire.
  *(Go n'utilise pas le préfixe `Get` : c'est `u.Name()`, pas `u.GetName()`.)*
- Constructeur `NewXxx` **seulement** si la construction exige quelque chose.
- Concevoir pour que la zéro-valeur soit utile.
- `String()` sur les types destinés à l'affichage ; jamais de `%v` sur soi-même dedans.
- Composition (embedding) plutôt que hiérarchies de types.
- Garder les structs petites et cohérentes : une struct de 20 champs cache généralement
  trois concepts distincts.

## Ce que je dois retenir

- Littéral **nommé** ; les champs omis prennent leur zéro-valeur.
- Une struct est **comparable** si tous ses champs le sont.
- **Récepteur pointeur** pour muter, pour les grosses structs, pour les types non copiables —
  et par **cohérence**.
- La conversion valeur/pointeur est automatique **sauf** sur une valeur non adressable (entrée de map).
- `String()` est appelée par `fmt` ; le type du récepteur détermine quand.
- L'**embedding n'est pas de l'héritage** : promotion de méthodes, sans polymorphisme.
- L'encapsulation passe par la **casse** des identifiants.

➡️ [Exercices](exercices.md)
