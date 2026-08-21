# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Aplatir

```go
func check(name string, age int, active bool) string {
	if name == "" {
		return "nom manquant"
	}
	if age < 18 {
		return "mineur"
	}
	if !active {
		return "compte inactif"
	}
	return "ok"
}
```

Quatre niveaux d'imbrication deviennent zéro. Les conditions d'échec se lisent en séquence et
le cas normal est la dernière ligne. Noter aussi que l'ordre des tests **change les messages**
sur une entrée doublement invalide : `check("", 10, false)` retourne « nom manquant », pas
« mineur ». Cet ordre est un choix, à assumer.

## E2 — IMC

```go
func BMICategory(weightKg, heightM float64) string {
	if heightM <= 0 {
		return "taille invalide"
	}
	bmi := weightKg / (heightM * heightM)
	switch {
	case bmi < 18.5:
		return "maigreur"
	case bmi < 25:
		return "normal"
	case bmi < 30:
		return "surpoids"
	default:
		return "obésité"
	}
}
```

En intervertissant `bmi < 25` et `bmi < 18.5`, **toute** valeur inférieure à 25 tombe dans le
premier cas : « maigreur » disparaît complètement. Le programme continue de fonctionner, sans
erreur, en donnant systématiquement une mauvaise réponse pour une plage entière. C'est
exactement le type de bug qu'un test aux bornes attraperait (niveau 5).

## E3 — Jours du mois

```go
func DaysInMonth(month, year int) (int, error) {
	switch month {
	case 1, 3, 5, 7, 8, 10, 12:
		return 31, nil
	case 4, 6, 9, 11:
		return 30, nil
	case 2:
		if isLeap(year) {
			return 29, nil
		}
		return 28, nil
	default:
		return 0, fmt.Errorf("mois invalide : %d", month)
	}
}

func isLeap(y int) bool {
	return y%4 == 0 && (y%100 != 0 || y%400 == 0)
}
```

| Année | Bissextile ? | Pourquoi |
|---|---|---|
| 1900 | **non** | divisible par 100 mais pas par 400 |
| 2000 | oui | divisible par 400 |
| 2024 | oui | divisible par 4, pas par 100 |
| 2100 | **non** | même cas que 1900 |

La règle « divisible par 4 » seule se trompe une fois par siècle. Elle a produit de vrais
incidents en production : plusieurs systèmes ont eu des bugs de date le 29 février 2000 et
2100 fera de même. `time.Date` de la bibliothèque standard gère cela correctement — à
préférer en production (niveau 4).

## E4 — `fallthrough`

```go
switch n {
case 1:
	fmt.Println("un")
	fallthrough
case 2:
	fmt.Println("deux")
case 3:
	fmt.Println("trois")
}
// n == 1 → affiche « un » PUIS « deux »
```

`fallthrough` exécute le cas suivant **sans réévaluer sa condition** : avec `n == 1`, on
affiche « deux » alors que `n != 2`. C'est ce qui le rend déroutant.

La version à valeurs multiples (`case 1, 2:`) est presque toujours plus lisible. `fallthrough`
reste justifié dans le cas rare où les cas sont **cumulatifs** : un niveau de permission qui
accorde tous les droits des niveaux inférieurs, par exemple. Il doit alors être commenté.

## E5 — Portée

Le code **ne compile pas** :

```
./main.go:8:14: undefined: v
```

`v` déclarée dans l'instruction d'initialisation du `if` n'existe que dans le `if` et ses
`else if`/`else`. Le dernier `fmt.Println(v)` est hors de cette portée. C'est précisément
l'intérêt de la construction : la variable ne survit pas là où elle n'a plus de sens.

## Exercice intermédiaire — `taxes`

```go
package main

import (
	"fmt"
	"os"
	"strconv"
)

// Barème PAR PART. Les seuils sont des bornes SUPÉRIEURES incluses :
// un revenu de 11 000 € pile est intégralement taxé à 0 %.
// Choix documenté, cohérent d'une tranche à l'autre.
const (
	seuil1, taux1 = 11_000.0, 0.00
	seuil2, taux2 = 28_000.0, 0.11
	seuil3, taux3 = 78_000.0, 0.30
	taux4         = 0.41
)

// Tax calcule l'impôt PAR TRANCHES et retourne aussi la tranche marginale.
// Fonction pure : aucun affichage, aucune sortie.
func Tax(income, parts float64) (amount, marginalRate float64, err error) {
	if income < 0 {
		return 0, 0, fmt.Errorf("revenu négatif : %.2f", income)
	}
	if parts <= 0 {
		return 0, 0, fmt.Errorf("nombre de parts invalide : %.2f", parts)
	}

	q := income / parts // quotient familial
	marginalRate = taux1

	perPart := 0.0
	if q > seuil3 {
		perPart += (q - seuil3) * taux4
		marginalRate = taux4
	}
	if q > seuil2 {
		perPart += (min(q, seuil3) - seuil2) * taux3
		if marginalRate == taux1 {
			marginalRate = taux3
		}
	}
	if q > seuil1 {
		perPart += (min(q, seuil2) - seuil1) * taux2
		if marginalRate == taux1 {
			marginalRate = taux2
		}
	}
	return perPart * parts, marginalRate, nil
}

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintln(os.Stderr, "usage: taxes <revenu> <parts>")
		os.Exit(1)
	}
	income, err := strconv.ParseFloat(os.Args[1], 64)
	if err != nil {
		fmt.Fprintf(os.Stderr, "revenu invalide : %v\n", err)
		os.Exit(1)
	}
	parts, err := strconv.ParseFloat(os.Args[2], 64)
	if err != nil {
		fmt.Fprintf(os.Stderr, "parts invalides : %v\n", err)
		os.Exit(1)
	}

	amount, rate, err := Tax(income, parts)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	fmt.Printf("revenu imposable : %.0f €\n", income)
	fmt.Printf("parts            : %.1f\n", parts)
	fmt.Printf("quotient familial: %.0f €\n", income/parts)
	fmt.Printf("tranche marginale: %.0f %%\n", rate*100)
	fmt.Printf("impôt            : %.0f €\n", amount)
	if income > 0 {
		fmt.Printf("taux moyen       : %.2f %%\n", amount/income*100)
	}
}
```

**Contrainte 1 — le calcul par tranches.** L'erreur universelle est d'écrire :
```go
if q > 28000 { return q * 0.30 }   // FAUX
```
Sur 28 001 €, cela donnerait 8 400 € d'impôt au lieu d'environ 1 870 €. Seule la **fraction**
du revenu au-dessus du seuil est taxée au taux supérieur. La conséquence, souvent mal comprise
du grand public : **gagner un euro de plus ne peut jamais faire baisser le revenu net**.

**Le piège caché — le seuil exact.** À 11 000 € pile, faut-il taxer ? Ici la condition est
`q > seuil1`, donc non : la borne est incluse dans la tranche inférieure. Ce qui compte n'est
pas le choix, mais qu'il soit **le même aux quatre seuils**. Une solution qui utilise `>` à un
seuil et `>=` à un autre crée une discontinuité d'un euro, invisible en test et fausse en
production.

**Contrainte 7 — aucun `else`.** Chaque `if` ajoute une contribution indépendante et les
conditions ne sont pas exclusives : c'est ce qui permet de s'en passer.

## Défi

**a) Machine à états**

```go
type State int

const (
	Rouge State = iota
	Vert
	Orange
)

func Next(s State) State {
	switch s {
	case Rouge:
		return Vert
	case Vert:
		return Orange
	case Orange:
		return Rouge
	default:
		return Rouge // état inconnu : on repart d'un état sûr
	}
}

func Seconds(s State) int {
	switch s {
	case Rouge:
		return 60
	case Vert:
		return 45
	case Orange:
		return 5
	default:
		return 0
	}
}
```

**b) Le piège de l'exhaustivité**

Ajouter `Clignotant` **compile parfaitement**. À l'exécution, `Next(Clignotant)` tombe dans le
`default` et retourne `Rouge` : la machine à états perd silencieusement un état. Sans
`default`, la fonction retournerait la zéro-valeur `State(0)` — soit `Rouge` également, mais
par accident.

Ce que cela révèle : **les énumérations Go ne sont pas des types somme.** `State` est un `int`
déguisé ; `State(42)` est une valeur parfaitement légale que le compilateur ne questionne pas.
Rust vérifie l'exhaustivité d'un `match` sur un `enum` et refuse de compiler si un variant
manque ; Java a des `enum` qui sont de vraies classes fermées. Go n'offre ni l'un ni l'autre —
c'est le prix de sa simplicité.

Parades : le linter `exhaustive` (inclus dans `golangci-lint`) signale les `switch` incomplets
sur un type énuméré, et un `default` qui **panique** ou journalise rend le trou visible en test
plutôt qu'invisible en production.

**c) Trois implémentations**

| | `switch` | table associative | tableau indexé |
|---|---|---|---|
| Lisibilité | excellente | bonne | moyenne (il faut décoder les indices) |
| Coût | quelques comparaisons | hachage + indirection | un accès mémoire |
| Valeur invalide | `default` explicite | clé absente → zéro-valeur silencieuse | **panique** (hors bornes) |
| Ajout d'un état | une ligne, mais dans chaque `switch` | une ligne, en un seul endroit | une ligne, en un seul endroit |

Jusqu'à une dizaine d'états, le `switch` gagne : il est explicite, sans allocation, et le
`default` traite le cas invalide. Au-delà, ou quand la table doit être modifiée à l'exécution,
la table associative devient préférable — la donnée est alors centralisée en un seul endroit
au lieu d'être dispersée dans plusieurs `switch` à maintenir cohérents.

Le tableau indexé est le plus rapide mais le plus fragile : il suppose que les valeurs de
l'énumération sont contiguës et commencent à 0, hypothèse qu'un `iota + 1` casse déjà.

## Réponses du quiz

1. **Non**, et `gofmt` les retire. La condition n'est pas parenthésée en Go.
2. **Non.** Go exige un `bool`. Ce choix supprime les `if (x = 5)` accidentels et les
   conversions implicites vers booléen, source de bugs dans d'autres langages.
3. `v` n'existe que dans le `if` et ses branches `else`/`else if`.
4. **Non** : Go ne « tombe » pas d'un cas au suivant. Le `break` implicite supprime le bug
   classique du C.
5. Il exécute le cas **suivant** sans réévaluer sa condition.
6. Un `switch` sans expression teste des conditions booléennes ; il remplace avantageusement
   trois `else if` ou plus.
7. **Dans l'ordre d'écriture** : le premier cas vrai gagne.
8. Parce qu'il ajoute un niveau d'indentation sans rien apporter : le `return` a déjà quitté
   la fonction. C'est l'une des remarques les plus fréquentes en revue de code Go.
9. Oui pour plusieurs valeurs (`case 1, 2, 3:`). **Non** pour une plage : Go n'a pas de
   `case 1..5`. Il faut un `switch` sans expression avec `case n >= 1 && n <= 5:`.
10. Oui, `goto` existe, restreint à la fonction courante. Non, on ne l'utilise pas : les
    labels sur les boucles couvrent les besoins légitimes.
