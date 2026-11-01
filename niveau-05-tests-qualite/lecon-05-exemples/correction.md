# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Premier exemple

```go
func ExampleSlug() {
	fmt.Println(slug.Slug("Go 1.27 : quoi de neuf ?"))
	// Output: go-1-27-quoi-de-neuf
}
```

En cas d'écart, `go test` affiche :

```
--- FAIL: ExampleSlug (0.00s)
got:
go-1-27-quoi-de-neuf
want:
go-127-quoi-de-neuf
```

Sans `// Output:`, l'exemple n'apparaît **pas** dans `go test -v` : il est compilé (une erreur de
compilation ferait échouer `go test`), mais jamais exécuté. C'est la seule façon de le voir : il
manque dans la liste des `=== RUN`.

## E2 — Les noms

```go
func Example()                {}  // paquet
func ExampleMedian()          {}  // Median
func ExampleMinMax_ints()     {}  // MinMax, suffixe « ints »
func ExampleMinMax_strings()  {}  // MinMax, suffixe « strings »
```

`ExampleMedian_Pair` : `go vet` répond `ExampleMedian_Pair refers to unknown field or method:
Median.Pair` — avec une majuscule, `Pair` est lu comme une **méthode** de `Median`.
`ExampleMediane` : `ExampleMediane refers to unknown identifier: Mediane`. Les deux fautes
sont détectées ; c'est une raison de plus de lancer `go vet` (que `go test` lance d'ailleurs en
partie automatiquement).

## E3 — `go doc -ex`

```bash
go doc -ex slices.SortFunc            # liste : ExampleSortFunc_caseInsensitive, ExampleSortFunc_multiField
go doc -ex slices.ExampleSortFunc_multiField
go doc -ex errors.As                  # ExampleAs
go doc -ex time.Duration.Round        # ExampleDuration_Round
```

L'exemple `ExampleSortFunc_multiField` montre exactement le patron `cmp.Or` de N3 L4 ;
`ExampleDuration_Round` montre l'arrondi à plusieurs précisions d'un coup. Réponse personnelle
pour « inconnu ».

## E4 — Sortie non ordonnée

Avec `// Output:` et `-count=10`, l'exemple échoue **certaines** fois seulement : l'ordre de la
map change. Avec `// Unordered output:`, il passe toujours. Mais la meilleure documentation est
celle qui montre la bonne pratique :

```go
func ExampleFrequencies() {
	freq := text.Frequencies("b a b c b a")
	for _, w := range slices.Sorted(maps.Keys(freq)) {
		fmt.Println(w, freq[w])
	}
	// Output:
	// a 2
	// b 3
	// c 1
}
```

Le lecteur apprend en passant qu'il faut trier pour afficher une map — ce qu'un
`Unordered output` lui aurait caché.

## E5 — pkg.go.dev en local

Les exemples apparaissent dans une section repliable « Example » sous l'identifiant auquel ils
sont rattachés (et l'exemple de paquet en tête, sous la vue d'ensemble). Le commentaire placé
au-dessus de la fonction `Example` devient le **texte d'introduction** de l'exemple ; le corps
est affiché comme du code, et la sortie attendue en dessous.

## Exercice intermédiaire — Documenter un paquet pour un inconnu

```go
// Package duration analyse des durées écrites par des humains.
//
// Une durée est une suite d'au moins une composante <entier><unité>, sans espace
// entre les composantes :
//
//	durée      = composante { composante }
//	composante = entier unité
//	unité      = "j" | "h" | "min" | "s"
//
// Exemples valides : "2h", "45min", "1h30min", "2j". Les espaces en début et fin
// sont ignorés. Sont refusés : la chaîne vide (ErrEmpty), un signe moins
// (ErrNegative), une unité inconnue ou absente (ErrUnit), un nombre manquant
// (strconv.ErrSyntax).
//
//	d, err := duration.Parse("1h30min")
//	if err != nil {
//		return err
//	}
//	time.Sleep(d)
package duration
```

```go
func ExampleParse() {
	for _, s := range []string{"2h", "1h30min", "3sem"} {
		d, err := duration.Parse(s)
		if errors.Is(err, duration.ErrUnit) {
			fmt.Printf("%s : unité inconnue\n", s)
			continue
		}
		fmt.Printf("%s = %v\n", s, d)
	}
	// Output:
	// 2h = 2h0m0s
	// 1h30min = 1h30m0s
	// 3sem : unité inconnue
}

func ExampleParse_zero() {
	d, err := duration.Parse("0h")
	fmt.Println(d, err)
	// Output: 0s <nil>
}
```

- La **grammaire** dans le commentaire de paquet : un bloc indenté par une tabulation est rendu
  comme du code par `go doc` et pkg.go.dev (syntaxe des commentaires de doc depuis Go 1.19).
- Chaque sentinelle est rattachée à la situation qui la produit : l'appelant sait quoi tester.
- `Parse("0h")` rend `0s` sans erreur : c'est un **choix** ; l'exemple le rend contractuel. Si
  quelqu'un décide un jour de refuser les durées nulles, l'exemple échouera et forcera la
  discussion.

## Défi — L'exemple complet

```go
// example_retry_test.go
package retry_test

import (
	"errors"
	"fmt"

	"exemple/retry"
)

type flakyService struct{ calls int }

func (s *flakyService) Call() error {
	s.calls++
	if s.calls < 3 {
		return errors.New("indisponible")
	}
	return nil
}

func Example() {
	svc := &flakyService{}
	err := retry.Retry(5, svc.Call)
	fmt.Println(err, "après", svc.calls, "appels")
	// Output: <nil> après 3 appels
}
```

D'après `go doc testing` : un fichier d'exemples est présenté comme un **exemple complet**
(« whole file example ») s'il contient **un seul** exemple, au moins une autre déclaration
(fonction, type, variable, constante) de niveau fichier, et **aucune** fonction de test ni de
benchmark. Ici : un seul `Example`, le type `flakyService` et sa méthode. Noter `svc.Call` passé
comme **valeur de méthode** (N2 L3) là où une `func() error` est attendue.

## Quiz

1. `func ExampleXxx()` : ni paramètre ni résultat.
2. Que la sortie standard correspond au commentaire `// Output:` (espaces de bord ignorés).
3. Il est compilé mais pas exécuté.
4. `func ExampleQueue_Push()`.
5. Sinon il est interprété comme une méthode : `ExampleT_Suffixe` désignerait la méthode `Suffixe`.
6. Quand l'ordre de la sortie n'est pas déterministe — en sachant qu'un exemple trié est souvent
   une meilleure documentation.
7. Pour que le lecteur voie les appels avec le nom du paquet, comme dans son propre code.
8. Elle inclut les exemples dans la sortie de `go doc` : liste pour un identifiant, programme
   complet pour un exemple nommé.
9. Non : il documente un usage typique ; les tests couvrent les cas limites et les erreurs.
10. Un fichier contenant un seul exemple et des déclarations auxiliaires, sans test ni benchmark ;
    affiché comme un programme entier.
