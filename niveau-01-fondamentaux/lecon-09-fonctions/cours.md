# Leçon 9 — Fonctions

## Objectifs

1. Déclarer des fonctions et comprendre que **tout est passé par valeur** en Go.
2. Utiliser les retours multiples et les résultats nommés à bon escient.
3. Écrire des fonctions variadiques et connaître leur piège.
4. Maîtriser la portée lexicale et le masquage (*shadowing*).

## Explication

### Signature : le nom avant le type

```go
func add(a int, b int) int       { return a + b }
func add(a, b int) int           { return a + b }   // types groupés
func divmod(a, b int) (int, int) { return a / b, a % b }
func greet(msg string)           { fmt.Println(msg) }  // aucun résultat
```

Le nom précède le type (`a int`, pas `int a`) : c'est l'inverse du C, et c'est délibéré —
une déclaration se lit de gauche à droite, y compris pour les types complexes.

Quelques faits :
- Pas de **surcharge** : deux fonctions ne peuvent pas partager un nom dans un package.
  C'est volontaire — savoir quelle fonction est appelée ne demande jamais de résoudre une
  surcharge mentalement.
- Pas d'**arguments par défaut**, pas d'arguments nommés à l'appel. L'équivalent idiomatique
  est le patron « options fonctionnelles », vu au niveau 3.
- La **récursivité** fonctionne normalement ; la pile d'une goroutine démarre petite (~8 Ko)
  et grandit dynamiquement, donc une récursion profonde est possible — jusqu'à une limite
  d'1 Go par défaut, au-delà de laquelle le programme s'arrête sur `stack overflow`.

### Tout est passé par valeur

Les arguments sont **copiés**. Copier un `int` est gratuit ; copier une struct de 200 octets
ne l'est pas.

```go
func double(n int) { n *= 2 }   // modifie une copie : sans effet chez l'appelant
```

Cas particulier essentiel : slices, maps et channels contiennent un **pointeur interne**.
Les copier duplique leur descripteur (quelques mots machine), pas leur contenu :

```go
func fill(s []int)              { s[0] = 99 }        // VISIBLE par l'appelant
func grow(s []int)              { s = append(s, 1) } // invisible : réaffecte la copie locale
func setKey(m map[string]int)   { m["k"] = 1 }       // VISIBLE
```

La distinction est celle de la leçon 6 : modifier le **contenu** partagé est visible,
remplacer le **descripteur** ne l'est pas. Les pointeurs (niveau 2) complètent le tableau.

### Retours multiples

```go
func divmod(a, b int) (int, int) {
	return a / b, a % b
}
q, r := divmod(17, 5)     // 3, 2
q, _ := divmod(17, 5)     // le « trou noir » _ ignore explicitement le second
```

Go autorise autant de résultats qu'on veut. Deux idiomes s'appuient dessus :

- **`(T, bool)`** — « valeur, présente ? ». C'est ce que produit `v, ok := m[k]` (leçon 7).
  On l'utilise quand l'absence est un cas **normal**, pas une anomalie.
- **`(T, error)`** — « valeur, ce qui a échoué ». C'est la convention centrale de Go, et
  elle remplace les exceptions. Elle fait l'objet de la **leçon 10**, juste après celle-ci.

Convention absolue : quand il y a une erreur, elle est **le dernier résultat**.

Au-delà de trois résultats, une struct nommée devient plus lisible qu'une liste de valeurs
anonymes — et elle survit mieux aux évolutions.

### Résultats nommés

```go
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return          // « naked return » : retourne x et y
}
```

Deux usages légitimes :
1. **Documenter** une signature ambiguë : `func find() (index int, found bool)` se lit mieux
   que `(int, bool)`. La documentation générée par `go doc` en profite.
2. Permettre à un `defer` de **modifier** la valeur retournée — technique vue au niveau 2.

Le `return` nu, en revanche, est déconseillé au-delà de quelques lignes : il oblige le
lecteur à remonter chercher ce qui est réellement retourné. Nommer les résultats, oui ;
retourner à vide dans une fonction de quarante lignes, non.

À noter : les résultats nommés sont initialisés à leur **zéro-valeur** dès l'entrée dans la
fonction.

### Fonctions variadiques

```go
func sum(nums ...int) int {
	total := 0
	for _, n := range nums {   // nums est un []int ordinaire
		total += n
	}
	return total
}

sum()               // 0 — nums est un slice nil, ce qui fonctionne parfaitement
sum(1, 2, 3)        // 6
xs := []int{1, 2, 3}
sum(xs...)          // 6 — « étalement » d'un slice existant
```

Points techniques :
- Le paramètre variadique doit être **le dernier** de la signature.
- À l'intérieur, c'est un slice. Sans argument, il vaut `nil` — et `range`, `len` et
  `append` fonctionnent sur un slice nil (leçon 6).
- **`sum(xs...)` ne copie pas `xs`** : la fonction reçoit un slice qui partage le même
  tableau sous-jacent. Si elle le modifie, l'appelant le voit. C'est le piège d’aliasing de
  la leçon 6 déguisé.
- `fmt.Println(a ...any)` est l'exemple canonique de la bibliothèque standard.

Une fonction variadique ne remplace **pas** des arguments optionnels : `f(a, b, opts ...int)`
où `opts` vaudrait « la précision » est illisible. Le bon outil sera le patron des options
fonctionnelles (niveau 3).

### Portée lexicale et masquage

```go
var global = "package"          // visible dans tout le package

func f() {
	x := "fonction"
	if true {
		x := "bloc"             // NOUVELLE variable qui masque la précédente
		fmt.Println(x)          // "bloc"
	}
	fmt.Println(x)              // "fonction"
}
```

Chaque paire d'accolades ouvre une portée. Une variable est visible de sa déclaration
jusqu'à la fin de son bloc. Le **masquage** (*shadowing*) est légal et parfois utile, mais
c'est une source de bugs classiques :

```go
data, err := load()
if err != nil { return err }

if cond {
	data, err := transform(data)   // ← masque LES DEUX ; le résultat est perdu
	_ = data
	_ = err
}
// ici, data est toujours la valeur d'origine
```

Le compilateur ne dit rien : les deux variables sont bien utilisées **à l'intérieur** du
bloc. Le linter `shadow` (via `golangci-lint`) le détecte ; `go vet` seul, non. La parade
est d'utiliser `=` et de déclarer `var err error` avant, ou de restructurer.

Règle de lecture : `:=` **déclare toujours au moins une nouvelle variable dans le bloc
courant**. Dans `a, err := f()`, si `a` est nouveau et `err` existe déjà dans **le même**
bloc, `err` est réaffecté — mais dans un bloc **imbriqué**, les deux sont neufs.

### Conception : ce qui distingue une bonne fonction

- **Une responsabilité.** Si le nom contient « And », c'est probablement deux fonctions.
- **Pas d'effet de bord caché.** Une fonction qui modifie son argument doit le dire dans son
  nom (`SortInPlace`) ou sa documentation.
- **Pure quand c'est possible.** Une fonction qui ne dépend que de ses paramètres est
  testable en une ligne (niveau 5) et sûre en concurrence (niveau 6).
- **Calcul séparé des entrées/sorties.** C'est le point le plus rentable de tout le niveau 1 :
  une fonction qui calcule *et* affiche est presque intestable.

## Exemple

```go
package main

import (
	"fmt"
	"strings"
)

// Compute retourne les statistiques d'un jeu de valeurs.
// Quatre résultats NOMMÉS : sans les noms, l'appelant devrait deviner l'ordre
// de (int, int, int, bool). Au niveau 2, on verra qu'une struct serait encore
// plus lisible au-delà de trois ou quatre résultats.
// Le dernier résultat suit l'idiome (T, bool) : un jeu vide n'est pas une
// anomalie, c'est simplement un cas sans statistiques.
func Compute(nums ...int) (count, lo, hi int, ok bool) {
	if len(nums) == 0 {
		return 0, 0, 0, false
	}
	lo, hi = nums[0], nums[0]
	for _, n := range nums {
		lo = min(lo, n) // min et max sont intégrés depuis Go 1.21
		hi = max(hi, n)
	}
	return len(nums), lo, hi, true
}

// Mean est séparée de Compute : une fonction, une responsabilité.
func Mean(nums ...int) (float64, bool) {
	if len(nums) == 0 {
		return 0, false
	}
	sum := 0
	for _, n := range nums {
		sum += n
	}
	return float64(sum) / float64(len(nums)), true // conversion AVANT la division
}

// indexOf illustre les résultats nommés à visée documentaire :
// (index int, found bool) se lit mieux que (int, bool).
func indexOf(words []string, target string) (index int, found bool) {
	for i, w := range words {
		if strings.EqualFold(w, target) {
			return i, true
		}
	}
	return -1, false // -1 : un indice impossible, donc inutilisable par accident
}

// pad ne modifie pas son entrée : elle retourne une nouvelle valeur.
func pad(s string, width int) string {
	if len(s) >= width {
		return s
	}
	return s + strings.Repeat(" ", width-len(s))
}

func main() {
	values := []int{7, 2, 9, 4}

	// Étalement d'un slice existant dans un paramètre variadique
	if n, lo, hi, ok := Compute(values...); ok {
		avg, _ := Mean(values...)
		fmt.Printf("%s n=%d min=%d max=%d moyenne=%.2f\n", pad("stats", 8), n, lo, hi, avg)
	}

	if _, _, _, ok := Compute(); !ok {
		fmt.Println("aucune valeur : pas de statistiques")
	}

	words := []string{"go", "Rust", "python"}
	if i, found := indexOf(words, "RUST"); found {
		fmt.Printf("trouvé à l'indice %d\n", i)
	}
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Compute(nums ...int) (count, lo, hi int, ok bool)` | Variadique **et** retours multiples nommés. Le `bool` dit « il y avait des données », ce qui n'est pas une erreur. |
| `Compute(values...)` | Étalement d'un slice. Attention : `nums` partage le tableau de `values`. |
| Quatre résultats nommés | À la limite du raisonnable : au-delà, une struct s'impose (niveau 2). Les noms tiennent lieu de documentation. |
| `Mean` séparée | Une fonction, une responsabilité. Elle recalcule la somme : c'est un coût assumé pour la clarté, et il serait mesuré avant d'être optimisé (niveau 11). |
| `float64(sum) / float64(len(nums))` | Conversion **avant** la division : `float64(sum/len(nums))` aurait déjà tronqué (leçon 3). |
| `(index int, found bool)` | Résultats nommés à visée documentaire. Le corps utilise quand même des `return` explicites. |
| `return -1, false` | `-1` est un indice impossible : même si l'appelant oublie de tester `found`, il ne l'utilisera pas par accident sans planter. |
| `pad` sans effet de bord | Retourne une nouvelle chaîne, n'en modifie aucune. Les chaînes sont immuables de toute façon (leçon 8). |
| `if _, _, _, ok := …` | Trois `_` : la valeur ne nous intéresse pas, seul le drapeau compte. Un peu lourd — encore un argument pour la struct du niveau 2. |

## Erreurs fréquentes

1. **Croire qu'une fonction peut modifier son paramètre valeur.** `func double(n int)` ne change rien chez l'appelant.
2. **Croire que `append` dans une fonction est visible** par l'appelant : non, le descripteur local est réaffecté.
3. **`f(xs...)` cru copiant** : il partage le tableau sous-jacent.
4. **Paramètre variadique ailleurs qu'en dernier** : ne compile pas.
5. **`return` nu dans une longue fonction** : le lecteur ne sait plus ce qui sort.
6. **Masquer une variable avec `:=` dans un bloc imbriqué** : bug silencieux, non détecté par `go vet`.
7. **Fonction qui calcule *et* affiche** : intestable, non réutilisable.
8. **Fonction trop longue.** Go n'impose pas de limite, mais au-delà d'une cinquantaine de lignes il y a presque toujours deux responsabilités.
9. **Chercher la surcharge** : elle n'existe pas. Il faut des noms distincts (`Parse`, `ParseWithBase`) ou des génériques (niveau 3).

## Bonnes pratiques Go

- Une fonction, une responsabilité, un nom qui la décrit sans « And ».
- Types groupés quand ils sont identiques : `func f(a, b, c int)`.
- Résultats nommés seulement quand ils clarifient ; `return` explicite dans le corps.
- Plus de trois résultats → une struct.
- Variadique pour un nombre réellement variable d'arguments, pas pour simuler des options.
- Documenter tout effet de bord, ou mieux : ne pas en avoir.
- Commentaire de documentation sur toute fonction exportée, commençant par son nom.
- Séparer systématiquement le calcul de l'affichage.

## Ce que je dois retenir

- **Tout est passé par valeur** ; slices, maps et channels copient un descripteur, pas leur contenu.
- Les **retours multiples** portent deux idiomes : `(T, bool)` et `(T, error)`.
- L'erreur, quand il y en a une, est **toujours le dernier résultat**.
- Le variadique est un slice à l'intérieur ; **`f(xs...)` partage** le tableau sous-jacent.
- Les résultats nommés documentent ; le `return` nu nuit dès que la fonction s'allonge.
- Le **masquage** par `:=` dans un bloc imbriqué est un bug silencieux et fréquent.

➡️ [Exercices](exercices.md)
