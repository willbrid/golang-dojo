# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Constante contre variable

```
constantes : true   0.29999999999999998890
variables  : false  0.30000000000000004441
```

Avec des constantes, le compilateur calcule `0.1+0.2` **exactement** (3/10), puis l'**arrondit
une seule fois** vers le `float64` le plus proche de 0,3 — qui est aussi celui qu'on obtient en
écrivant `0.3` : égalité. Avec des variables, `0.1` et `0.2` sont **déjà arrondis** chacun, et
leur somme est **arrondie à nouveau** : les deux erreurs s'accumulent et tombent sur le
`float64` voisin.

## E2 — Le tableau des arrondis

| x | `Round` | `RoundToEven` | `Floor` | `Ceil` | `Trunc` | `int(x)` | `%.0f` |
|---|---|---|---|---|---|---|---|
| 0.5 | 1 | 0 | 0 | 1 | 0 | 0 | 0 |
| 1.5 | 2 | 2 | 1 | 2 | 1 | 1 | 2 |
| -0.5 | -1 | -0 | -1 | -0 | -0 | 0 | -0 |
| -1.7 | -2 | -2 | -2 | -1 | -1 | -1 | -2 |

`%.0f` applique l'**arrondi au pair** (comme `RoundToEven`) : `0.5` → `0`, `1.5` → `2`. Et il
affiche `-0` : le zéro négatif existe et `fmt` le montre. `int(-0.5)` vaut `0` (les entiers
n'ont pas de zéro négatif).

## E3 — 2⁵³

Le plus petit `n` est **9 007 199 254 740 992 = 2⁵³**. Un `float64` a 52 bits de mantisse
stockés + 1 bit implicite = 53 bits de précision. Jusqu'à 2⁵³, tous les entiers sont
représentables ; au-delà, l'écart entre deux `float64` consécutifs vaut 2, puis 4, etc.
`2⁵³ + 1` tombe entre deux valeurs représentables et est arrondi à `2⁵³`.

Conséquence pratique : JavaScript n'a que des `float64` pour ses nombres. Un identifiant
`int64` supérieur à 2⁵³ envoyé en JSON à un navigateur revient **modifié**. D'où la pratique
d'envoyer les grands identifiants sous forme de **chaînes** (leçon 6, option `,string`).

## E4 — Jeton

```go
import (
	"crypto/rand"
	"math/big"
)

func ResetCode() (string, error) {
	n, err := rand.Int(rand.Reader, big.NewInt(100_000_000))
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("%08d", n.Int64()), nil
}
```

La version `math/rand/v2` n'est **pas acceptable**. Son générateur global (ChaCha8) est
certes initialisé aléatoirement, mais `math/rand` ne **promet** aucune propriété de sécurité :
la documentation dit explicitement de ne pas l'utiliser pour du travail sensible, et un autre
générateur (`NewPCG`) est carrément prévisible à partir de quelques sorties observées. Le
critère n'est pas « est-ce que ça a l'air aléatoire » mais « un attaquant peut-il prédire la
valeur suivante ». Seul `crypto/rand` le garantit.

`%08d` : le code doit garder ses zéros de tête (`00042137`).

## E5 — `sort.Search`

```go
stock := []int{50, 40, 22, 15, 9, 3}
i := sort.Search(len(stock), func(i int) bool { return stock[i] < 10 })   // 4
if i == len(stock) {
	// jamais passé sous 10
}
```

`sort.Search(n, f)` rend le plus petit `i` dans `[0, n)` tel que `f(i)` est vrai, ou **`n`**
si aucun ne l'est. Il suppose `f` monotone (faux… faux, vrai… vrai) — ici garanti par la
décroissance du stock.

## Exercice intermédiaire — Facture

```go
// roundDiv calcule a/b arrondi au plus proche, demis vers le haut (a ≥ 0, b > 0).
func roundDiv(a, b int64) int64 {
	return (2*a + b) / (2 * b)
}

func Compute(lines []Line) (Invoice, error) {
	inv := Invoice{Lines: lines}
	for i, l := range lines {
		if l.Quantity < 0 {
			return Invoice{}, fmt.Errorf("ligne %d (%q) : quantité négative %d", i+1, l.Label, l.Quantity)
		}
		if l.VATRate < 0 || l.VATRate > 10000 {
			return Invoice{}, fmt.Errorf("ligne %d (%q) : taux %d hors de [0, 10000]", i+1, l.Label, l.VATRate)
		}
		ht := l.UnitHT * Cents(l.Quantity)
		vat := Cents(roundDiv(int64(ht)*int64(l.VATRate), 10000))
		inv.TotalHT += ht
		inv.TotalVAT += vat
	}
	inv.TotalTTC = inv.TotalHT + inv.TotalVAT
	return inv, nil
}
```

Points de correction :

- **`roundDiv`** : `(2a + b) / (2b)` = `a/b + 1/2` tronqué, c'est-à-dire arrondi au plus proche,
  demis vers le haut, **en entiers**. Tester : `roundDiv(5, 10)` = 1, `roundDiv(4, 10)` = 0,
  `roundDiv(15, 10)` = 2. Attention : valable pour `a ≥ 0` ; pour un avoir (montant négatif),
  il faudrait traiter le signe à part.
- **Contrainte 4** : 3 × 0,99 = 2,97 € HT. TVA 5,5 % = 2,97 × 0,055 = 0,16335 € → **0,16 €**.
  TTC : **3,13 €**. En entiers : `297 × 550 = 163 350`, `/10000` → 16,335 centimes → 16.
- **Contrainte 5** : 9,00 € HT à 5,5 %. Valeur exacte : 9,495 € → arrondi demi vers le haut :
  **9,50 €**. En `float64` : `9.00 * 1.055` vaut `9.49499999999999922…`, arrondi à **9,49 €**.
  Un centime perdu, sans erreur ni avertissement, et la facture est fausse. Le cas se trouve en
  cherchant un montant dont la valeur exacte tombe **pile sur un demi-centime** : c'est
  précisément là qu'une erreur binaire minuscule change le sens de l'arrondi.
- **Débordement** : `ht × rate` en `int64` déborde au-delà de ~9 × 10¹⁴ centimes de HT par
  ligne. Largement suffisant pour une facture ; à documenter pour un système bancaire.
- Remarque métier : en France, l'arrondi de TVA peut se faire par ligne ou sur le total selon
  les cas. Le code **impose** une règle et la nomme : c'est ce qui compte.

## Défi — Simulation reproductible

⚠️ Le piège de ce défi est dans l'énoncé lui-même : `rand.N` est une **fonction du paquet**, qui
utilise le générateur **global**. Pour une simulation reproductible, il faut que **tous** les
tirages passent par `r`. Or `rand.N` est générique et n'a **pas** d'équivalent en méthode sur
`*rand.Rand`. On écrit donc `time.Duration(r.Int64N(int64(3 * time.Minute)))`.

Version correcte :

```go
func dur(r *rand.Rand, max time.Duration) time.Duration {
	return time.Duration(r.Int64N(int64(max)))
}

func simulate(r *rand.Rand, cashiers int) (avg, worst time.Duration, n int) {
	const day = 8 * time.Hour
	free := make([]time.Duration, cashiers)
	var total time.Duration
	for t := dur(r, 3*time.Minute); t < day; t += dur(r, 3*time.Minute) {
		i := slices.Index(free, slices.Min(free)) // la caisse libérée la plus tôt
		start := max(t, free[i])
		wait := start - t
		total += wait
		worst = max(worst, wait)
		free[i] = start + time.Minute + dur(r, 4*time.Minute)
		n++
	}
	return total / time.Duration(n), worst, n
}
```

Avec `r := rand.New(rand.NewPCG(42, 0))`, deux exécutions donnent la même sortie.

Pour les 100 simulations, l'algorithme de Welford calcule moyenne et variance **en un
passage**, sans soustraire deux grands nombres voisins (source de perte de précision) :

```go
var n int
var mean, m2 float64
for seed := range uint64(100) {
	avg, _, _ := simulate(rand.New(rand.NewPCG(seed, 0)), 1)
	x := avg.Minutes()
	n++
	d := x - mean
	mean += d / float64(n)
	m2 += d * (x - mean)
}
stddev := math.Sqrt(m2 / float64(n-1))
```

Mesuré sur 100 graines (0 à 99) :

| Caisses | Attente moyenne |
|---|---|
| 1 | 237 min ± 20 |
| 2 | 9 min ± 6 |

Avec **une** caisse, le temps de service moyen (3 min) dépasse l'intervalle moyen entre deux
arrivées (1,5 min) : la file **croît sans limite** au fil de la journée, et l'attente moyenne
approche 4 heures. Avec **deux** caisses, la capacité dépasse à peine la demande et l'attente
tombe à 9 minutes : une division par **26**, pas par 2. Le niveau 11, leçon 7 (files
d'attente) explique pourquoi l'attente explose quand le taux d'occupation approche 100 %.

## Quiz

1. En binaire, 0,1 a un développement infini (comme 1/3 en décimal) : il est arrondi.
2. Les constantes non typées sont calculées en précision arbitraire et arrondies une seule fois.
3. Avec une tolérance relative : `|a−b| ≤ eps × max(|a|, |b|)`.
4. `+Inf` pour le flottant ; **panique** (`integer divide by zero`) pour l'entier — et erreur de
   compilation si le diviseur est une constante.
5. `math.IsNaN(x)` (jamais `x == math.NaN()`, toujours faux).
6. Pour les demis : `Round` s'éloigne de zéro, `RoundToEven` va vers le pair.
7. En entiers de la plus petite unité (centimes), dans un type dédié ; `math/big` si nécessaire.
8. Pas de `Seed`, générateur global initialisé aléatoirement, noms clairs (`IntN`), `rand.N`
   générique, générateurs modernes (PCG, ChaCha8).
9. `rand.New(rand.NewPCG(graine1, graine2))`, et faire passer **tous** les tirages par lui.
10. Dès qu'un attaquant gagnerait à deviner la valeur : jetons, mots de passe, clés, nonces.
