# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Map nil

```go
var m map[string]int
_ = m["x"]      // OK  → 0
_ = len(m)      // OK  → 0
for range m {}  // OK  → 0 tour
delete(m, "x")  // OK  → ne fait rien
m["x"] = 1      // panic: assignment to entry in nil map
```

Seule **l'écriture** panique. L'asymétrie surprend, mais elle est logique : lire une table
inexistante peut retourner la zéro-valeur, écrire exigerait d'allouer la table — et Go
refuse d'allouer implicitement quelque chose que le développeur n'a pas demandé.

## E2 — Absent ou zéro

```go
m := map[string]int{"a": 0}

if v, ok := m["a"]; ok { fmt.Println("a existe, vaut", v) }  // existe, vaut 0
if _, ok := m["b"]; !ok { fmt.Println("b n'existe pas") }
```

`m["a"] == 0` et `m["b"] == 0` sont tous deux vrais : sans le `ok`, les deux cas sont
indiscernables.

## E3 — Ordre

```go
// Non déterministe
for k := range m { fmt.Print(k, " ") }

// Déterministe
for _, k := range slices.Sorted(maps.Keys(m)) { fmt.Print(k, " ") }
```

`maps.Keys` retourne un itérateur (`iter.Seq[K]`, Go 1.23+) ; `slices.Sorted` le consomme
et retourne un slice trié. Avant Go 1.23, il fallait une boucle `append` puis `slices.Sort`.

## E4 — Unique

```go
func Unique(words []string) []string {
	seen := make(map[string]struct{}, len(words))
	out := make([]string, 0, len(words))
	for _, w := range words {
		if _, ok := seen[w]; ok {
			continue
		}
		seen[w] = struct{}{}
		out = append(out, w) // l'ORDRE vient du slice, pas de la map
	}
	return out
}
```

Le point de l'exercice : la map sert uniquement de **test d'appartenance**. L'ordre est
porté par le parcours du slice d'entrée. Construire le résultat en itérant sur la map
donnerait un ordre aléatoire.

## E5 — Invert

```go
// Invert retourne une map valeur→clé.
// Choix documenté : en cas de valeurs dupliquées, la DERNIÈRE clé rencontrée
// l'emporte. Comme l'ordre d'itération d'une map est aléatoire, le résultat
// est alors NON DÉTERMINISTE — ce qui est presque toujours indésirable.
func Invert(m map[string]int) map[int]string { … }

// Alternative sans perte, et déterministe :
func InvertMulti(m map[string]int) map[int][]string {
	out := make(map[int][]string, len(m))
	for k, v := range m {
		out[v] = append(out[v], k)
	}
	for _, ks := range out {
		slices.Sort(ks) // rétablit le déterminisme
	}
	return out
}
```

`out[v] = append(out[v], k)` sur une clé absente fonctionne : `out[v]` vaut `nil`, et
`append` sur un slice nil alloue. Idiome très fréquent.

## Exercice intermédiaire — `index`

```go
package main

import (
	"bufio"
	"fmt"
	"maps"
	"os"
	"slices"
	"strings"
	"unicode"
)

// Index associe chaque mot aux numéros de ligne où il apparaît.
// Choix documenté : une ligne n'est listée QU'UNE FOIS par mot, même si le mot
// y apparaît plusieurs fois — c'est le comportement attendu d'un index de
// consultation. Le comptage des occurrences est un autre besoin, un autre outil.
type Index map[string][]int

func normalize(w string) string {
	return strings.ToLower(strings.TrimFunc(w, func(r rune) bool {
		return !unicode.IsLetter(r) && !unicode.IsDigit(r)
	}))
}

// Build lit ligne par ligne : la mémoire utilisée est celle de l'index,
// jamais celle du fichier.
func Build(f *os.File, minLen int) (Index, error) {
	idx := make(Index)
	sc := bufio.NewScanner(f)
	sc.Buffer(make([]byte, 0, 64*1024), 4*1024*1024)

	for lineNo := 1; sc.Scan(); lineNo++ {
		seenOnLine := make(map[string]struct{})
		for _, raw := range strings.Fields(sc.Text()) {
			w := normalize(raw)
			if len([]rune(w)) < minLen {
				continue
			}
			if _, dup := seenOnLine[w]; dup {
				continue
			}
			seenOnLine[w] = struct{}{}
			idx[w] = append(idx[w], lineNo)
		}
	}
	if err := sc.Err(); err != nil {
		return nil, fmt.Errorf("lecture : %w", err)
	}
	return idx, nil
}

// Print n'effectue aucun calcul ; Build n'effectue aucun affichage.
func (idx Index) Print(w *os.File) {
	for _, word := range slices.Sorted(maps.Keys(idx)) {
		lines := make([]string, 0, len(idx[word]))
		for _, n := range idx[word] {
			lines = append(lines, fmt.Sprint(n))
		}
		fmt.Fprintf(w, "%-12s : %s\n", word, strings.Join(lines, ", "))
	}
}
```

**Sur la contrainte 1** — pourquoi `map[string][]int` et non `map[string]map[int]bool` ?
Parce que les numéros de ligne arrivent **déjà triés** (on lit le fichier dans l'ordre) : un
slice les conserve triés gratuitement, occupe 8 octets par entrée contre ~50 pour une map,
et se parcourt séquentiellement. La map ne serait justifiée que si l'on devait tester
l'appartenance en O(1) ou insérer dans le désordre. Ici, ni l'un ni l'autre.

**Sur la contrainte 7** — le fichier n'est jamais chargé entièrement, mais **l'index, lui,
tient en mémoire**. Sur 500 Mo de texte, il peut peser plusieurs centaines de mégaoctets.
C'est la vraie limite de cette conception, et l'énoncer est plus important que la
contourner : un index de cette taille se stocke sur disque (c'est ce que fait un moteur de
recherche).

## Défi

**a) Regroupements**

```go
func GroupByFirstLetter(words []string) map[string][]string {
	out := make(map[string][]string)
	for _, w := range words {
		if w == "" {
			continue
		}
		r := []rune(w)[0] // et non w[0], qui donnerait un demi-caractère accentué
		key := strings.ToLower(string(r))
		out[key] = append(out[key], w)
	}
	return out
}

func GroupByLength(words []string) map[int][]string {
	out := make(map[int][]string)
	for _, w := range words {
		n := len([]rune(w)) // des CARACTÈRES, pas des octets
		out[n] = append(out[n], w)
	}
	return out
}

// GroupByAnagram : la clé canonique d'un anagramme est le mot dont les lettres
// ont été TRIÉES. Deux mots sont anagrammes si et seulement si leurs lettres
// triées sont identiques — la fonction de clé transforme une relation
// d'équivalence en une valeur comparable.
func GroupByAnagram(words []string) map[string][]string {
	out := make(map[string][]string)
	for _, w := range words {
		r := []rune(strings.ToLower(w))
		slices.Sort(r)
		key := string(r)
		out[key] = append(out[key], w)
	}
	return out
}
```

`out[key] = append(out[key], w)` sur une clé absente fonctionne : `out[key]` vaut `nil`, et
`append` sur un slice nil alloue. C'est l'idiome de regroupement le plus fréquent en Go.

Les trois fonctions ne diffèrent **que par le calcul de la clé** : c'est exactement ce que la
leçon 11 permettra de factoriser en passant une fonction en paramètre.

**b) Opérations d'ensemble**

```go
func Union(a, b map[string]bool) map[string]bool {
	out := make(map[string]bool, len(a)+len(b))
	maps.Copy(out, a) // Go 1.21+
	maps.Copy(out, b)
	return out
}

// Intersection itère sur LE PLUS PETIT des deux ensembles.
// Coût : O(min(|a|,|b|)) au lieu de O(|a|). Sur 10 éléments contre 1 000 000,
// c'est 100 000 fois moins de tours de boucle pour le même résultat.
func Intersection(a, b map[string]bool) map[string]bool {
	if len(b) < len(a) {
		a, b = b, a
	}
	out := make(map[string]bool)
	for k := range a {
		if b[k] {
			out[k] = true
		}
	}
	return out
}

func Difference(a, b map[string]bool) map[string]bool { // a \ b
	out := make(map[string]bool)
	for k := range a {
		if !b[k] {
			out[k] = true
		}
	}
	return out
}

func IsSubset(a, b map[string]bool) bool { // a ⊆ b ?
	if len(a) > len(b) {
		return false // court-circuit gratuit
	}
	for k := range a {
		if !b[k] {
			return false
		}
	}
	return true
}
```

Noter que `Difference` **ne peut pas** échanger les arguments : elle n'est pas symétrique.
L'optimisation de `Intersection` ne fonctionne que parce que l'intersection l'est.

**c) TopN déterministe**

```go
func TopN(counts map[string]int, n int) []string {
	words := slices.Collect(maps.Keys(counts))
	slices.SortFunc(words, func(a, b string) int {
		if c := cmp.Compare(counts[b], counts[a]); c != 0 {
			return c // fréquence décroissante
		}
		return cmp.Compare(a, b) // départage alphabétique : LE point crucial
	})
	return words[:min(n, len(words))]
}
```

Sans le départage alphabétique, deux mots de même fréquence sortent dans un ordre qui dépend
de l'itération de la map — donc **différent à chaque exécution**. Sur vingt lancements avec des
ex æquo, on obtient plusieurs sorties distinctes.

C'est le scénario exact du test qui passe cent fois en local et échoue une fois en intégration
continue, sans qu'on ait rien changé. Le déterminisme n'est pas une coquetterie : c'est une
propriété qu'on décide d'avoir ou de subir.

`cmp.Compare` est préférable à la soustraction `counts[b] - counts[a]` : la soustraction peut
déborder sur de très grands entiers, et le tri devient alors incohérent.

**d) Coût mémoire**

| Structure | Mémoire pour 0..1 000 000 | Rapport |
|---|---|---|
| `[]bool` | ~1 Mo (1 octet par entrée) | référence |
| `map[int]bool` | ~50 à 90 Mo | ×50 à ×90 |

Une map stocke, pour chaque entrée, la clé, la valeur, un octet de contrôle, et maintient un
facteur de charge inférieur à 1 — d'où le surcoût. Le `[]bool` n'a aucun surcoût par entrée et
se parcourt séquentiellement, ce qui le rend aussi bien plus rapide.

**Quand le `[]bool` cesse d'être le bon choix :** dès que le domaine des valeurs est
**creux**. Un ensemble contenant `{1, 7, 2000000000}` demanderait un `[]bool` de deux
gigaoctets, contre trois entrées dans une map. Le critère est le rapport entre le nombre
d'éléments et l'étendue des valeurs possibles — pas le nombre d'éléments seul.

## Réponses du quiz

1. La **zéro-valeur** du type de valeur — ici `0`. Aucune erreur, aucune panique.
2. Seule **l'écriture** panique. Lecture, `len`, `range` et `delete` sont sûrs.
3. Il indique si la clé **existe**, ce qui lève l'ambiguïté avec la zéro-valeur.
4. **Par conception.** Le runtime randomise le point de départ pour empêcher tout code de
   dépendre d'un ordre que la spécification ne garantit pas.
5. Les types non comparables : slices, maps, fonctions — et toute struct qui en contient un.
   Les interfaces compilent mais paniquent à l'exécution si la valeur dynamique n'est pas
   comparable.
6. Non. Les entrées d'une map ne sont **pas adressables** : le runtime peut les déplacer
   lors d'un redimensionnement, un pointeur deviendrait invalide.
7. `fatal error: concurrent map writes` — une erreur **fatale**, non récupérable par
   `recover`. Le runtime la détecte délibérément plutôt que de corrompre la table.
8. Rien. `delete` sur une clé absente est une opération valide sans effet.
9. Avec une map dont la valeur ne sert pas : `map[T]bool` de préférence, parce que `if set[k]`
   se lit directement. La variante `map[T]struct{}` n'occupe aucun octet par valeur, mais
   l'économie ne compte qu'au-delà de plusieurs millions d'entrées.
10. **Oui.** Une map se comporte comme une référence : la fonction reçoit un descripteur qui
    pointe vers la même table.
