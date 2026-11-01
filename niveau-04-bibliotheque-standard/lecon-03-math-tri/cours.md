# Leçon 3 — Math, aléatoire, tri

> Trois sujets réunis par un même thème : **les nombres ne se comportent pas comme en
> mathématiques**. Un flottant n'est pas un réel, un générateur aléatoire n'est pas forcément
> imprévisible, et un tri dépend d'une comparaison qui doit être cohérente. Chaque partie de
> cette leçon correspond à une famille de bugs de production.

## Objectifs

1. Comprendre pourquoi `0.1 + 0.2 != 0.3`, et comparer des flottants correctement.
2. Connaître les valeurs spéciales (`Inf`, `NaN`, `-0`) et les fonctions d'arrondi.
3. Ne jamais représenter de l'argent avec un `float64`.
4. Tirer des nombres avec `math/rand/v2`, et savoir quand il faut `crypto/rand`.
5. Situer le paquet `sort` par rapport à `slices`.

## Explication

### Un `float64` est une fraction **binaire**

Un `float64` (IEEE 754) stocke un nombre sous la forme *mantisse × 2^exposant*, avec 53 bits
de mantisse. Tout nombre qui n'est pas une somme finie de puissances de 2 est **arrondi** :
`0.5` est exact, `0.1` ne l'est pas — en binaire, c'est `0.0001100110011…` à l'infini, comme
`1/3` en décimal.

```go
a, b := 0.1, 0.2
fmt.Println(a+b == 0.3, a+b)   // false 0.30000000000000004
```

Et pourtant :

```go
fmt.Println(0.1+0.2 == 0.3)    // true !
```

Rappel du niveau 1, leçon 2 : les **constantes** non typées sont calculées en précision
arbitraire par le compilateur. `0.1+0.2` est calculé **exactement**, puis arrondi une seule
fois. Avec des variables, chaque opérande est déjà arrondi, et l'addition arrondit encore.
Ce qui compile pareil ne calcule pas pareil.

### Comparer des flottants

On ne compare jamais deux flottants **calculés** avec `==`. On compare leur écart à une
tolérance :

```go
func almostEqual(a, b, eps float64) bool {
	return math.Abs(a-b) <= eps*math.Max(math.Abs(a), math.Abs(b))
}
```

Tolérance **relative** (proportionnelle à la grandeur des nombres) : une tolérance absolue de
`1e-9` est énorme pour des valeurs autour de `1e-12` et dérisoire autour de `1e12`. Le choix
de `eps` dépend du calcul ; `1e-9` est un point de départ courant.

Autre limite : au-delà de 2⁵³ (≈ 9 × 10¹⁵), un `float64` ne représente plus tous les entiers.
`float64(1<<53) + 1 == float64(1<<53)`. Un identifiant `int64` qui transite par un `float64`
(par exemple via JSON en JavaScript, leçon 6) peut être **silencieusement modifié**.

### Les valeurs spéciales

| Expression (avec des variables) | Résultat |
|---|---|
| `1 / 0.0` | `+Inf` |
| `-1 / 0.0` | `-Inf` |
| `0.0 / 0.0`, `math.Sqrt(-1)` | `NaN` |
| `math.Copysign(0, -1)` | `-0` (et `-0 == 0` est vrai) |

Une division **entière** par zéro panique ; une division **flottante** par zéro rend `±Inf`
sans erreur. Et NaN est contagieux : toute opération qui le touche rend NaN, et toute
comparaison avec lui est fausse (`NaN == NaN` est faux). On le teste avec `math.IsNaN(x)`,
`math.IsInf(x, 0)`.

Un `NaN` qui entre dans une moyenne ou un total **pollue tout** sans lever d'erreur : valider
les entrées flottantes à la frontière.

### Arrondir

| Fonction | `2.5` | `-2.5` | Règle |
|---|---|---|---|
| `math.Round` | 3 | -3 | au plus proche, les demis **loin de zéro** |
| `math.RoundToEven` | 2 | -2 | au plus proche, les demis vers le **pair** (« arrondi bancaire ») |
| `math.Floor` | 2 | -3 | vers −∞ |
| `math.Ceil` | 3 | -2 | vers +∞ |
| `math.Trunc` / `int(x)` | 2 | -2 | vers zéro |

Et `fmt` ? `%.2f` arrondit la **valeur binaire réelle** : `fmt.Printf("%.2f", 2.675)` affiche
`2.67`, parce que `2.675` est stocké comme `2.67499999…`. Ce n'est pas un bug de `fmt`.

### L'argent ne se compte pas en `float64`

```go
sum := 0.0
for range 10 {
	sum += 0.1
}
fmt.Println(sum, sum == 1.0) // 0.9999999999999999 false
```

Dix centimes ajoutés dix fois ne font pas un euro. Sur des millions de transactions, l'écart
devient visible dans une comptabilité. Les règles :

- **Montants en entiers** de la plus petite unité : `int64` de centimes. `1234` = 12,34 €.
- Les arrondis (TVA, partage d'une addition) se font **explicitement**, à un endroit, avec une
  règle documentée (et le centime restant attribué à quelqu'un).
- Pour des calculs exacts sur des fractions : `math/big.Rat`. Pour de très grands entiers :
  `math/big.Int`. Lents, mais exacts.

### Débordement des entiers

Rappel du niveau 1 : un débordement entier est **silencieux** (`int8(127) + 1` sur des
variables vaut `-128`). Le paquet `math/bits` fournit des opérations qui rendent la retenue :

```go
sum, carry := bits.Add64(math.MaxUint64, 1, 0)   // 0, 1  → la retenue signale le débordement
hi, lo := bits.Mul64(x, y)                        // produit sur 128 bits
```

Pour les entiers signés, on teste avant d'opérer (c'est le défi `SafeAdd` du niveau 1).

### `math/rand/v2` : l'aléatoire de tous les jours

Go 1.22 a introduit `math/rand/v2`, à utiliser dans tout code neuf. Différences avec l'ancien
`math/rand` :

- **Plus de `Seed` à appeler** : le générateur global est initialisé aléatoirement au démarrage.
  (Avec l'ancien paquet avant Go 1.20, oublier `rand.Seed` donnait **la même séquence** à chaque
  exécution — un bug célèbre.)
- Noms plus clairs : `rand.IntN(n)` dans `[0, n)`, `rand.Float64()` dans `[0, 1)`,
  `rand.Shuffle`, `rand.Perm`.
- Une fonction générique : `rand.N(10 * time.Second)` rend une `time.Duration` aléatoire
  dans `[0, 10s)` — pratique pour les délais avec gigue (*jitter*, niveau 10).

Pour une séquence **reproductible** (tests, simulations), on crée son propre générateur avec
une graine fixe :

```go
r := rand.New(rand.NewPCG(1, 2))   // même graine → même séquence, à chaque exécution
r.IntN(100)
```

### `crypto/rand` : quand l'imprévisibilité compte

`math/rand/v2` est **rapide** et statistiquement bon, mais **prévisible** pour un attaquant
qui observe des sorties. Pour tout ce qui touche à la sécurité — jetons de session, mots de
passe générés, clés, identifiants de réinitialisation, nonces — on utilise `crypto/rand` :

```go
token := rand.Text()   // crypto/rand, Go 1.24 : 26 caractères base32, 128 bits d'aléa
```

Règle simple : **si une personne malveillante gagne quelque chose à deviner la valeur,
`crypto/rand`**. Les deux paquets s'appellent `rand` : importer l'un sous un alias
(`mrand "math/rand/v2"`) quand on a besoin des deux.

### `sort` et `slices`

La leçon 4 du niveau 3 a présenté `slices.Sort` et `slices.SortFunc`. Le paquet `sort`
reste présent dans beaucoup de code existant, sous trois formes :

| Forme | Exemple | Statut |
|---|---|---|
| fonctions par type | `sort.Ints`, `sort.Strings` | remplacées par `slices.Sort` |
| `sort.Slice(s, less)` | comparateur sur **indices** | remplacé par `slices.SortFunc` |
| `sort.Interface` (`Len`, `Less`, `Swap`) | pour trier une structure qui **n'est pas** un slice | toujours utile, rare |
| `sort.Search(n, f)` | recherche dichotomique **sur un prédicat** | utile : trouve le premier `i` où `f(i)` est vrai |

`sort.Search` mérite d'être connu : il ne cherche pas une **valeur** mais la frontière d'un
prédicat monotone (« le premier jour où le stock passe sous 10 », « la plus petite taille de
lot qui tient en mémoire »).

### Trier des flottants

`slices.Sort` sur des `float64` place les NaN **en tête** (ordre de `cmp.Compare`) :

```go
x := []float64{3, math.NaN(), 1, math.Inf(-1)}
slices.Sort(x)   // [NaN -Inf 1 3]
```

C'est cohérent et déterministe — mais si des NaN se trouvent dans les données, c'est en
général le signe d'un problème en amont.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

// Cents représente un montant en centimes : jamais de float64 pour de l'argent.
type Cents int64

func (c Cents) String() string {
	sign := ""
	if c < 0 {
		sign, c = "-", -c
	}
	return fmt.Sprintf("%s%d,%02d €", sign, c/100, c%100)
}

// Split partage un montant entre n personnes. Les centimes restants sont
// attribués aux premières parts : la somme des parts est EXACTEMENT le total.
func Split(total Cents, n int) ([]Cents, error) {
	if n <= 0 {
		return nil, fmt.Errorf("partage en %d parts", n)
	}
	base, rest := total/Cents(n), total%Cents(n)
	parts := make([]Cents, n)
	for i := range parts {
		parts[i] = base
		if Cents(i) < rest {
			parts[i]++
		}
	}
	return parts, nil
}

// Mean refuse les valeurs non finies plutôt que de rendre un NaN silencieux.
func Mean(xs []float64) (float64, error) {
	if len(xs) == 0 {
		return 0, errors.New("aucune valeur")
	}
	var sum float64
	for i, x := range xs {
		if math.IsNaN(x) || math.IsInf(x, 0) {
			return 0, fmt.Errorf("valeur %d non finie : %v", i, x)
		}
		sum += x
	}
	return sum / float64(len(xs)), nil
}

func almostEqual(a, b, eps float64) bool {
	return math.Abs(a-b) <= eps*math.Max(math.Abs(a), math.Abs(b))
}

func main() {
	parts, _ := Split(10000, 3) // 100 € entre 3
	var check Cents
	for _, p := range parts {
		check += p
	}
	fmt.Println(parts, "total :", check)

	x, y := 0.1, 0.2
	fmt.Println(x+y == 0.3, almostEqual(x+y, 0.3, 1e-9))

	zero := 0.0
	if _, err := Mean([]float64{1, 2, zero / zero}); err != nil {
		fmt.Println("erreur :", err)
	}

	// Générateur à graine fixe : sortie identique à chaque exécution.
	r := rand.New(rand.NewPCG(2026, 11))
	deck := []string{"A♠", "K♥", "Q♦", "J♣", "10♠"}
	r.Shuffle(len(deck), func(i, j int) { deck[i], deck[j] = deck[j], deck[i] })
	fmt.Println(deck, r.IntN(6)+1)

	temps := []float64{21.5, math.NaN(), 19, 23.25}
	slices.Sort(temps)
	fmt.Println(temps, math.Round(-2.5), math.RoundToEven(-2.5))
}
```

Sortie :

```
[33,34 € 33,33 € 33,33 €] total : 100,00 €
false true
erreur : valeur 2 non finie : NaN
[A♠ 10♠ J♣ Q♦ K♥] 5
[NaN 19 21.5 23.25] -3 -2
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `type Cents int64` | Le type porte l'unité : impossible de confondre avec un nombre d'euros. |
| `String()` et le signe | `c/100` et `c%100` sur un négatif donneraient `-1,-50` : on traite le signe d'abord (modulo signé, niveau 1, leçon 3). |
| `Split` et `rest` | La règle d'attribution du reste est **explicite**. Vérification : la somme des parts vaut exactement 100,00 €. |
| `Mean` refuse NaN/Inf | Une donnée invalide est signalée à la frontière au lieu de contaminer le résultat. |
| `zero / zero` | Avec une constante `0.0/0.0`, le compilateur refuse (`division by zero`) : il faut une variable pour fabriquer un NaN. |
| `rand.NewPCG(2026, 11)` | Graine fixe : la sortie de cet exemple est reproductible, on peut la recopier. Sans graine (`rand.Shuffle` global), elle changerait à chaque exécution. |
| `math.Round(-2.5)` → `-3` | Demis loin de zéro ; `RoundToEven` → `-2`. |

## Erreurs fréquentes

1. **`==` entre flottants calculés.**
2. **De l'argent en `float64`.**
3. **Une tolérance absolue** pour des nombres de grandeurs très différentes.
4. **Un `int64` qui passe par un `float64`** et perd ses derniers chiffres.
5. **Ne pas tester NaN** à l'entrée d'un calcul.
6. **`math/rand` pour un jeton de sécurité.**
7. **Oublier la graine fixe** dans un test qui dépend de l'aléatoire, et obtenir un test
   instable.
8. **`int(x)` en pensant arrondir** : c'est une troncature vers zéro.
9. **Partager un montant par division flottante** et perdre (ou créer) un centime.

## Bonnes pratiques Go

- Montants en entiers de la plus petite unité, dans un type dédié.
- Tolérance relative pour comparer des flottants ; valider les entrées (`IsNaN`, `IsInf`).
- `math/rand/v2` pour l'aléatoire courant, `crypto/rand` pour tout ce qui est secret.
- Générateur à graine fixe pour les tests et simulations reproductibles.
- `slices` pour trier ; `sort.Search` pour chercher la frontière d'un prédicat.
- `math/big` quand l'exactitude prime sur la vitesse.

## Ce que je dois retenir

- `0.1` n'est pas représentable exactement en binaire ; les constantes, elles, sont calculées exactement.
- Comparer les flottants avec une tolérance **relative**.
- Division flottante par zéro → `±Inf` ; `0/0` → `NaN` ; `NaN != NaN`.
- `Round` (demis loin de zéro), `RoundToEven`, `Floor`, `Ceil`, `Trunc` = `int(x)`.
- L'argent en **centimes entiers**.
- `math/rand/v2` : pas de `Seed`, `IntN`, `N`, `NewPCG` pour la reproductibilité.
- **`crypto/rand`** dès qu'il y a un enjeu de sécurité ; `rand.Text()` pour un jeton.

---

➡️ [Exercices](exercices.md)
