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

**a) GroupBy**

```go
func GroupBy(words []string, key func(string) string) map[string][]string {
	out := make(map[string][]string)
	for _, w := range words {
		out[key(w)] = append(out[key(w)], w)
	}
	return out
}

// Par première lettre
GroupBy(words, func(w string) string { return string([]rune(w)[0]) })
// Par longueur
GroupBy(words, func(w string) string { return strconv.Itoa(len([]rune(w))) })
// Par anagramme : LA clé canonique est le mot avec ses lettres TRIÉES
GroupBy(words, func(w string) string {
	r := []rune(strings.ToLower(w))
	slices.Sort(r)
	return string(r)
})
```

Le cas anagramme est le vrai exercice : deux mots sont anagrammes **si et seulement si**
leurs lettres triées sont identiques. La fonction de clé transforme une relation
d'équivalence en une valeur comparable — c'est le patron général du regroupement.

**b) LRU**

Une map seule ne suffit pas : elle donne l'accès en O(1) mais ne dit pas quel élément est le
plus ancien. Il faut y ajouter une structure qui maintient **l'ordre d'usage** et permet de
déplacer un élément en tête en O(1) — une **liste doublement chaînée**. La map stocke alors
`clé → pointeur vers le nœud`.

```go
type entry struct {
	key   string
	value int
	prev  *entry
	next  *entry
}

type LRU struct {
	capacity int
	items    map[string]*entry
	head     *entry // le plus récemment utilisé
	tail     *entry // le plus ancien : la victime de l'éviction
}

func (c *LRU) Get(key string) (int, bool) {
	e, ok := c.items[key]
	if !ok {
		return 0, false
	}
	c.moveToFront(e) // un Get COMPTE comme un usage
	return e.value, true
}

func (c *LRU) Put(key string, value int) {
	if e, ok := c.items[key]; ok {
		e.value = value
		c.moveToFront(e)
		return
	}
	if len(c.items) == c.capacity {
		delete(c.items, c.tail.key) // évincer AVANT d'insérer
		c.remove(c.tail)
	}
	e := &entry{key: key, value: value}
	c.items[key] = e
	c.pushFront(e)
}
```

Les deux erreurs classiques : oublier que `Get` rafraîchit l'ordre, et oublier de retirer la
clé de la **map** en même temps que le nœud de la liste (fuite mémoire silencieuse : la map
grossit indéfiniment).

`container/list` de la stdlib fait le travail de la liste, mais l'écrire à la main une fois
apprend beaucoup plus. Version production : y ajouter un `sync.Mutex` (niveau 4) et des
métriques de taux de succès (niveau 9).

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
9. `struct{}` n'occupe aucun octet, mais `map[string]bool` se lit mieux (`if set[k]`).
   Préférer `bool` sauf enjeu mémoire mesuré.
10. **Oui.** Une map se comporte comme une référence : la fonction reçoit un descripteur qui
    pointe vers la même table.
