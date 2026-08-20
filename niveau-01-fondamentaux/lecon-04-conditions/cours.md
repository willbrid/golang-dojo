# Leçon 4 — Conditions et branchements

## Objectifs

1. Écrire un `if` idiomatique, avec instruction d'initialisation.
2. Pratiquer le *early return*, la forme de code Go par excellence.
3. Maîtriser le `switch`, qui ne ressemble à celui d'aucun autre langage courant.

## Explication

### `if` : pas de parenthèses, accolades obligatoires

```go
if x > 10 {
	fmt.Println("grand")
} else if x > 5 {
	fmt.Println("moyen")
} else {
	fmt.Println("petit")
}
```

Pas de parenthèses autour de la condition ; accolades **obligatoires** même pour une seule
instruction — ce qui élimine d'emblée le bug du `if` sans accolades suivi de deux
instructions. La condition doit être un `bool` : `if x { … }` avec `x` entier ne compile
pas, et `if x = 5` non plus puisqu'une affectation n'est pas une expression.

L'accolade ouvrante doit rester sur la même ligne, conséquence de l'insertion automatique
des points-virgules (leçon 1).

### L'instruction d'initialisation : l'idiome le plus courant de Go

```go
if v, ok := m[key]; ok {
	fmt.Println(v)
}   // v et ok n'existent plus ici

if err := doSomething(); err != nil {
	return err
}

if n := len(s); n > 100 {
	return fmt.Errorf("trop long : %d", n)
}
```

La variable déclarée avant le `;` n'existe que dans le `if` et ses `else`. C'est l'idiome le
plus fréquent du langage : il **limite la portée** au strict nécessaire, ce qui évite le
masquage accidentel et signale au lecteur que la valeur n'est pas utile plus loin.

### *Early return* : la forme de code Go par excellence

```go
// À éviter : imbrication en escalier
func process(u *User) error {
	if u != nil {
		if u.Active {
			if err := save(u); err == nil {
				return nil
			} else {
				return err
			}
		} else {
			return errors.New("inactif")
		}
	}
	return errors.New("nil")
}

// Idiomatique : les cas d'erreur sortent tôt, le chemin normal reste à gauche
func process(u *User) error {
	if u == nil {
		return errors.New("utilisateur nil")
	}
	if !u.Active {
		return errors.New("utilisateur inactif")
	}
	return save(u)
}
```

La règle, formulée dans *Effective Go* : **le chemin heureux reste aligné à gauche**, les
cas d'erreur sortent immédiatement. On lit la fonction en diagonale sans dérouler
mentalement une pile de `else`. Un `else` après un `return` est presque toujours du bruit —
c'est l'une des remarques les plus fréquentes en revue de code Go.

### `switch` : sans `break`, et beaucoup plus puissant

```go
switch day {
case "samedi", "dimanche":     // plusieurs valeurs par cas
	fmt.Println("week-end")
case "lundi":
	fmt.Println("courage")
default:
	fmt.Println("semaine")
}
```

Différences majeures avec C, Java ou JavaScript :

- **Pas de `break` à écrire** : Go ne « tombe » pas d'un cas au suivant. L'oubli de `break`,
  bug classique en C, n'existe pas.
- Pour forcer le passage au cas suivant : `fallthrough` — rare, et il faut le justifier.
  Il passe au cas suivant **sans réévaluer sa condition**.
- Un cas peut lister **plusieurs valeurs**.
- Le `default` peut se placer n'importe où, mais la convention est de le mettre en dernier.

**Le switch sans expression** remplace avantageusement les cascades de `if / else if` :

```go
switch {
case score >= 90:
	grade = "A"
case score >= 80:
	grade = "B"
default:
	grade = "F"
}
```

Les cas sont évalués **dans l'ordre** : le premier vrai gagne. C'est pourquoi
`score >= 90` doit précéder `score >= 80`.

Il accepte aussi une instruction d'initialisation :

```go
switch hour := time.Now().Hour(); {
case hour < 12:
	return "bonjour"
default:
	return "bonsoir"
}
```

Enfin, `switch` s'utilise sur n'importe quel type **comparable**, y compris des types
personnalisés — et il existe une forme spéciale, le **switch de type**
(`switch v := x.(type)`), réservée aux interfaces et vue au niveau 2.

### `goto` : il existe, on ne s'en sert pas

Go possède `goto`, restreint à la fonction courante et interdit de sauter par-dessus une
déclaration de variable. En pratique, on ne le rencontre presque que dans du code généré et
dans quelques boucles très chaudes de la bibliothèque standard. Le connaître suffit ; les
labels sur les boucles (leçon 5) couvrent les besoins légitimes.

## Exemple

Aucune boucle ni collection ici : elles arrivent aux leçons suivantes.

```go
package main

import (
	"fmt"
	"os"
	"strconv"
)

// classify range une note dans une mention.
// Un switch sans expression remplace cinq if/else if.
func classify(score int) string {
	switch {
	case score < 0 || score > 100:
		return "invalide"
	case score >= 90:
		return "excellent"
	case score >= 75:
		return "bien"
	case score >= 50:
		return "passable"
	default:
		return "insuffisant"
	}
}

// grade valide puis convertit — early return systématique, aucun else.
func grade(raw string) (string, error) {
	if raw == "" {
		return "", fmt.Errorf("note vide")
	}
	n, err := strconv.Atoi(raw)
	if err != nil {
		return "", fmt.Errorf("note %q : %w", raw, err)
	}
	if n < 0 || n > 100 {
		return "", fmt.Errorf("note %d hors des bornes [0,100]", n)
	}
	return classify(n), nil
}

// season utilise un switch à plusieurs valeurs par cas.
func season(month int) string {
	switch month {
	case 12, 1, 2:
		return "hiver"
	case 3, 4, 5:
		return "printemps"
	case 6, 7, 8:
		return "été"
	case 9, 10, 11:
		return "automne"
	default:
		return "mois invalide"
	}
}

// triangle classe un triangle à partir de ses côtés.
// L'ordre des cas est porteur de sens : on rejette d'abord l'impossible.
func triangle(a, b, c int) string {
	switch {
	case a <= 0 || b <= 0 || c <= 0:
		return "côtés invalides"
	case a+b <= c || a+c <= b || b+c <= a:
		return "inégalité triangulaire non respectée"
	case a == b && b == c:
		return "équilatéral"
	case a == b || b == c || a == c:
		return "isocèle"
	default:
		return "scalène"
	}
}

func main() {
	if g, err := grade(os.Args[1]); err != nil {
		fmt.Fprintf(os.Stderr, "refusé : %v\n", err)
		os.Exit(1)
	} else {
		// Ici un else est justifié : les deux branches font des choses différentes,
		// et g n'existe que dans la portée du if.
		fmt.Printf("mention : %s\n", g)
	}

	fmt.Println(season(7))
	fmt.Println(triangle(3, 4, 5))
	fmt.Println(triangle(2, 2, 9))

	// Instruction d'initialisation : m n'existe que dans le if.
	if m := 13; season(m) == "mois invalide" {
		fmt.Printf("%d n'est pas un mois\n", m)
	}
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `switch {` sans expression | Remplace une cascade de `if/else if`. Les cas sont testés **dans l'ordre** : intervertir deux lignes change le résultat. |
| Le cas de rejet en premier | Placer les cas invalides en tête est un réflexe : le reste de la fonction peut alors supposer une entrée valide. |
| `grade` en *early return* | Trois conditions d'échec, trois sorties, aucune imbrication. |
| `%w` | Enveloppe l'erreur de `strconv` pour ne pas perdre sa cause. Approfondi en leçon 10. |
| `case 12, 1, 2:` | Plusieurs valeurs par cas, sans `break` ni `fallthrough`. |
| Le `else` de `main` | Contre-exemple assumé : il est acceptable ici parce que `g` n'existe que dans la portée du `if`. Après un `return`, en revanche, il serait du bruit. |
| `if m := 13; …` | Instruction d'initialisation : `m` disparaît après le `if`. |

## Erreurs fréquentes

1. **Mettre des parenthèses** : `if (x > 10) {` compile mais `gofmt` les retire. Ce n'est pas du Go.
2. **Écrire `break` à la fin de chaque `case`** : inutile. Réflexe importé du C.
3. **`else` après un `return`** : bruit visuel, systématiquement signalé en revue.
4. **Ordre des cas d'un `switch` sans expression** : mettre `score >= 50` avant `score >= 90` fait que personne n'atteint jamais « excellent ».
5. **Oublier le `default`** dans un `switch` sur une énumération : les valeurs hors énumération passent silencieusement.
6. **Croire que `if x` fonctionne** avec un entier ou une chaîne : Go exige un `bool`.
7. **Condition trop longue** : `if a && b || c && !d` est illisible. Extraire dans une fonction nommée ou des variables intermédiaires.
8. **`fallthrough` par habitude** : il ne réévalue pas la condition du cas suivant, ce qui surprend.

## Bonnes pratiques Go

- *Early return* systématique ; garder le chemin normal aligné à gauche.
- `if v, ok := …; ok` pour limiter la portée des variables temporaires.
- `switch` sans expression dès qu'il y a trois branches ou plus.
- Toujours un `default` sur un `switch` qui produit une valeur — même s'il ne fait que
  signaler l'inattendu.
- Nommer les conditions complexes : `if isEligible(u) && hasQuota(u)` plutôt qu'une
  expression de trois lignes.
- Ne pas utiliser `goto`.

## Ce que je dois retenir

- `if` sans parenthèses, accolades obligatoires, condition **strictement booléenne**.
- L'instruction d'initialisation (`if v, ok := …; ok`) limite la portée : idiome n°1 de Go.
- **Early return** : les erreurs sortent tôt, le cas normal reste à gauche, pas d'`else`.
- `switch` **n'a pas** de `break` implicite à écrire ; plusieurs valeurs par cas ; il
  fonctionne sans expression.
- Les cas d'un `switch` sans expression sont évalués **dans l'ordre**.

➡️ [Exercices](exercices.md)
