# Leçon 3 — Méthodes

## Objectifs

1. Déclarer des méthodes et comprendre ce qu'est un **récepteur**.
2. Choisir entre **récepteur valeur** et **récepteur pointeur** — la décision la plus
   structurante de ce niveau.
3. Comprendre l'**ensemble de méthodes** d'un type, notion qui expliquera tout au chapitre
   des interfaces.
4. Implémenter `String()` et écrire des constructeurs idiomatiques.

## Explication

### Une méthode est une fonction avec un récepteur

```go
type User struct {
	Name, Email string
}

func (u User) FullName() string {          // récepteur VALEUR
	return u.Name + " <" + u.Email + ">"
}

func (u *User) SetName(name string) {      // récepteur POINTEUR
	u.Name = name
}
```

Le récepteur s'écrit entre `func` et le nom. Techniquement, `u.FullName()` est équivalent à
`User.FullName(u)` — c'est bien une fonction dont le premier paramètre a une place
particulière.

Deux règles à connaître d'emblée :

- **On ne peut définir des méthodes que sur un type déclaré dans le même package.**
  Impossible d'ajouter une méthode à `string`, à `int` ou à `time.Time`. Mais on peut
  déclarer `type MyString string` et lui en donner autant qu'on veut. Cette restriction
  évite les conflits du *monkey patching* d'autres langages : le comportement d'un type ne
  peut jamais être modifié à distance.
- **Un type nommé de n'importe quelle sorte peut avoir des méthodes**, pas seulement une
  struct : `type Celsius float64`, `type IDs []int`, `type Handler func(int)`,
  `type Counters map[string]int`.

Le nom du récepteur est court par convention — une ou deux lettres, cohérentes sur tout le
type (`u` pour `User`, jamais `this` ni `self`).

### Valeur ou pointeur ? La décision centrale

```go
func (u User) Rename(n string)  { u.Name = n }   // modifie une COPIE : sans effet
func (u *User) Rename(n string) { u.Name = n }   // modifie l'original
```

Le premier compile parfaitement et ne fait rien. C'est le **bug silencieux le plus fréquent**
de ce niveau : aucun avertissement, aucune erreur, juste un comportement absent.

**Récepteur pointeur si :**
1. la méthode **modifie** le récepteur ;
2. la struct est **volumineuse** et la copier coûterait (à mesurer, pas à supposer) ;
3. le type contient un `sync.Mutex` ou un autre élément qu'on ne doit pas copier ;
4. **la cohérence l'exige** — voir juste en dessous.

**Récepteur valeur si :**
1. le type est petit et immuable par nature (`time.Time`, `Point`, `Celsius`) ;
2. la méthode ne modifie rien.

**La règle de cohérence est la plus importante :** si **une seule** méthode d'un type a un
récepteur pointeur, **toutes** devraient en avoir un. Mélanger les deux crée des surprises
avec les interfaces et déroute le lecteur. En cas de doute : **récepteur pointeur partout**.
C'est le conseil du wiki officiel Go.

### L'ensemble de méthodes (*method set*)

C'est la notion qui rend tout le reste compréhensible :

| Type | Ensemble de méthodes |
|---|---|
| `T` | les méthodes à récepteur **valeur** |
| `*T` | les méthodes à récepteur **valeur** *et* **pointeur** |

Autrement dit, `*T` sait tout faire, `T` non. C'est asymétrique, et c'est la raison
technique de la règle de cohérence. Les conséquences apparaîtront pleinement à la leçon 4
(interfaces) : un `T` ne satisfera pas une interface dont une méthode a un récepteur
pointeur, avec ce message déroutant :

```
User does not implement Namer (method SetName has pointer receiver)
```

### L'appel automatique — et sa limite

Go convertit automatiquement entre valeur et pointeur lors d'un appel de méthode :

```go
u := User{}
u.SetName("x")              // Go réécrit (&u).SetName("x")

p := &User{}
fmt.Println(p.FullName())   // Go réécrit (*p).FullName()
```

Cela ne fonctionne que si la valeur est **adressable**. D'où l'échec classique :

```go
users := map[string]User{"a": {}}
users["a"].SetName("x")     // ERREUR : les entrées de map ne sont pas adressables
```

Il faut soit une `map[string]*User`, soit lire / modifier / réécrire. C'est le retour du
point vu au niveau 1, leçon 7, et une source réelle de frustration.

Note importante : **une méthode à récepteur pointeur appelée sur un pointeur `nil` ne
panique pas en soi.** Elle ne panique que si elle déréférence le récepteur. On peut donc
écrire délibérément des méthodes qui tolèrent `nil` :

```go
func (l *List) Len() int {
	if l == nil {
		return 0
	}
	return l.size
}
```

### `String()` : l'interface `fmt.Stringer`

```go
func (u User) String() string {
	return fmt.Sprintf("User(%s)", u.Name)
}

fmt.Println(u)              // User(Alice) — appelée automatiquement
fmt.Printf("%v / %s\n", u, u)
```

`fmt` teste à l'exécution si la valeur satisfait `fmt.Stringer` (`String() string`) et
l'appelle. C'est un premier contact concret avec les interfaces : aucune déclaration n'est
nécessaire, la présence de la méthode suffit.

**Deux pièges :**

1. **Ne jamais utiliser `%v` ou `%s` sur le récepteur *dans* `String()`** — récursion
   infinie et `fatal error: stack overflow`. Utiliser les champs directement, ou convertir
   vers le type sous-jacent (`float64(c)`).
2. **Le type du récepteur détermine ce qui l'active.** Avec `func (u *User) String()`,
   `fmt.Println(u)` sur une **valeur** n'appellera pas la méthode : elle affichera la struct
   brute, sans erreur ni avertissement. Bug silencieux.

Le même mécanisme existe pour `Error() string` (interface `error`) et
`MarshalJSON() ([]byte, error)` (niveau 4).

### Constructeurs

Go n'a pas de constructeurs. La convention est une fonction `NewXxx` :

```go
func NewUser(name, email string) (*User, error) {
	if name == "" {
		return nil, errors.New("nom vide")
	}
	return &User{Name: name, Email: email}, nil
}
```

Elle n'est nécessaire que si la construction demande une **validation**, une
**initialisation** (map, channel, slice préalloué) ou une ressource. **Si la zéro-valeur
suffit, ne pas écrire de constructeur.**

Conventions de nommage : `NewUser` dans un package `models`, mais **`New`** tout court si le
package ne construit qu'un type — d'où `list.New()`, `bytes.NewBuffer()`,
`errors.New()`. Le nom du package fait partie du nom : `models.NewUser` est bien,
`models.NewModelsUser` est du bégaiement.

### Accesseurs

Go n'utilise **pas** le préfixe `Get` :

```go
func (a *Account) Balance() int64      // ✔  a.Balance()
func (a *Account) GetBalance() int64   // ✘  non idiomatique
func (a *Account) SetBalance(v int64)  // ✔  le préfixe Set, lui, se conserve
```

Et surtout : **ne pas écrire d'accesseur qui ne fait qu'exposer un champ**. Si le champ n'a
aucun invariant à protéger, l'exporter directement. Un accesseur se justifie quand il valide,
calcule, ou protège un accès concurrent.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

// Account représente un compte bancaire. Le solde est en CENTIMES :
// jamais de flottant pour de l'argent (niveau 1, leçon 2).
type Account struct {
	ID      string
	Owner   string
	balance int64 // non exporté : l'invariant « jamais négatif » est protégé
}

var ErrInsufficientFunds = errors.New("fonds insuffisants")

// NewAccount valide et construit. Un constructeur est justifié : il y a une
// validation, et l'invariant doit être vrai dès la première ligne de vie de l'objet.
func NewAccount(id, owner string, initial int64) (*Account, error) {
	if strings.TrimSpace(id) == "" {
		return nil, errors.New("identifiant vide")
	}
	if initial < 0 {
		return nil, fmt.Errorf("solde initial négatif : %d", initial)
	}
	return &Account{ID: id, Owner: owner, balance: initial}, nil
}

// Balance ne modifie rien, mais garde un récepteur pointeur PAR COHÉRENCE
// avec Deposit et Withdraw. Mélanger les deux serait le vrai défaut.
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

// String satisfait fmt.Stringer. On n'utilise PAS %v sur a : ce serait une
// récursion infinie.
func (a *Account) String() string {
	return fmt.Sprintf("%s (%s) : %d,%02d €", a.ID, a.Owner, a.balance/100, a.balance%100)
}

// Celsius montre qu'un type non-struct peut avoir des méthodes,
// et que le récepteur valeur est ici le bon choix : le type est petit et immuable.
type Celsius float64

func (c Celsius) Fahrenheit() Celsius { return c*9/5 + 32 }
func (c Celsius) String() string      { return fmt.Sprintf("%.1f°C", float64(c)) }

func main() {
	acc, err := NewAccount("FR001", "Alice", 10_000)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(acc) // String() appelée : acc est déjà un *Account

	if err := acc.Withdraw(15_000); err != nil {
		fmt.Println("refusé :", err)
		fmt.Println("cause connue ?", errors.Is(err, ErrInsufficientFunds))
	}

	_ = acc.Deposit(2_500)
	fmt.Println(acc)

	t := Celsius(21.5)
	fmt.Printf("%v = %.1f°F\n", t, float64(t.Fahrenheit()))
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `balance int64` non exporté | L'invariant « le solde ne peut pas devenir négatif » tient **parce que** personne hors du package ne peut écrire ce champ. L'encapsulation en Go passe par la casse, pas par des mots-clés. |
| `func (a *Account) Balance()` | Ne modifie rien, mais reste pointeur : **cohérence**. |
| `%w` sur `ErrInsufficientFunds` | L'appelant peut tester la cause avec `errors.Is` tout en lisant un message riche (leçon 6). |
| `String()` sur `*Account` | `fmt.Println(acc)` fonctionne car `acc` est un `*Account`. Sur une valeur `Account`, la méthode ne serait **pas** appelée. |
| `float64(c)` dans `Celsius.String()` | Convertir vers le type sous-jacent évite la récursion infinie. |
| `Celsius` avec récepteur valeur | Type petit et immuable : le pointeur n'apporterait rien et compliquerait l'usage. |
| `10_000` | Séparateur de milliers, autorisé dans les littéraux numériques. |

## Erreurs fréquentes

1. **Récepteur valeur alors qu'on veut modifier** : la méthode s'exécute sans effet, sans erreur.
2. **Mélanger récepteurs valeur et pointeur** sur un même type.
3. **`%v` sur le récepteur dans `String()`** → `stack overflow`.
4. **`String()` à récepteur pointeur** puis affichage d'une valeur : la méthode n'est pas appelée, silencieusement.
5. **Appeler une méthode à récepteur pointeur sur une entrée de map** → ne compile pas.
6. **Écrire un constructeur inutile** quand la zéro-valeur suffit.
7. **Préfixe `Get`** sur les accesseurs.
8. **Accesseurs mécaniques** sur tous les champs : si rien n'est protégé, autant exporter le champ.
9. **`this` ou `self` comme nom de récepteur** : non idiomatique.
10. **Méthode sur un type d'un autre package** : ne compile pas — il faut un type local.

## Bonnes pratiques Go

- **Cohérence des récepteurs** ; en cas de doute, pointeur partout.
- Nom de récepteur court et identique dans toutes les méthodes du type.
- Constructeur `NewXxx` **seulement** si la construction exige quelque chose ; `New` tout
  court quand le package ne construit qu'un type.
- Champs non exportés dès qu'un invariant existe ; sinon, champs exportés sans accesseur.
- `String()` sur les types destinés à l'affichage ; jamais de `%v` sur soi-même dedans.
- Méthodes courtes : au-delà d'une trentaine de lignes, extraire.
- Documenter les méthodes exportées en commençant par leur nom.
- Une méthode qui ne dépend pas du récepteur devrait être une **fonction**, pas une méthode.

## Ce que je dois retenir

- Une méthode est une fonction avec un **récepteur** ; on n'en définit que sur des types
  **du package courant**.
- N'importe quel type nommé peut avoir des méthodes, pas seulement une struct.
- **Ensemble de méthodes** : `T` n'a que les récepteurs valeur, `*T` a les deux.
- **Récepteur pointeur** pour muter, pour les gros types, pour les types non copiables, et
  par **cohérence**.
- La conversion valeur/pointeur est automatique **sauf** sur une valeur non adressable.
- `String()` est appelée par `fmt` ; le type du récepteur détermine quand.
- Pas de préfixe `Get`, pas d'accesseur qui ne protège rien.

➡️ [Exercices](exercices.md)
