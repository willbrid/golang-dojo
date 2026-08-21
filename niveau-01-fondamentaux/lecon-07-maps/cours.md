# Leçon 7 — Maps

## Objectifs

1. Créer, lire, écrire et supprimer dans une map en toute sûreté.
2. Maîtriser l'idiome `v, ok := m[k]` et savoir quand il est indispensable.
3. Comprendre pourquoi l'ordre d'itération est aléatoire, et comment produire une sortie déterministe.
4. Connaître les contraintes sur les clés, et le comportement d'une map `nil`.

## Explication

### Création

```go
var m map[string]int          // nil ! lecture OK, ÉCRITURE = PANIQUE
m1 := make(map[string]int)    // vide et utilisable
m2 := make(map[string]int, 100) // avec une taille estimée : évite des réallocations
m3 := map[string]int{         // littéral
	"alice": 30,
	"bob":   25,             // virgule finale OBLIGATOIRE
}
```

La map `nil` est un piège asymétrique, à retenir absolument :

```go
var m map[string]int
fmt.Println(m["x"])    // 0     — lecture : OK
fmt.Println(len(m))    // 0     — OK
for k := range m { }   //       — OK, zéro tour
delete(m, "x")         //       — OK, ne fait rien
m["x"] = 1             // PANIQUE : assignment to entry in nil map
```

Contrairement au slice `nil` sur lequel `append` fonctionne, on ne peut **pas** écrire dans
une map `nil`. C'est la première cause de panique chez les débutants, en général via une map
qu'on a déclarée sans l'initialiser :

```go
var counts map[string]int   // nil
counts["a"]++               // PANIQUE : assignment to entry in nil map
```

Toujours créer la map avant d'y écrire — avec `make` ou un littéral. Au niveau 2, on verra
que le cas le plus fréquent est une map **champ d'un type** qu'on a oublié d'initialiser à la
construction.

### Lecture : la zéro-valeur masque l'absence

```go
m := map[string]int{"alice": 30}
fmt.Println(m["bob"])   // 0 — pas d'erreur, pas de panique
```

Lire une clé absente retourne la **zéro-valeur** du type de valeur. Impossible donc de
distinguer « bob a 0 » de « bob n'existe pas »… sauf avec l'idiome à deux résultats :

```go
if age, ok := m["bob"]; ok {
	fmt.Println("trouvé :", age)
} else {
	fmt.Println("absent")
}
```

C'est la « virgule ok ». Elle est indispensable dès que la zéro-valeur est une valeur
légitime. Pour un `map[string]bool` utilisé en ensemble, `m[k]` seul suffit et se lit mieux.

### Écriture, suppression, taille

```go
m["carol"] = 35        // ajoute ou remplace
delete(m, "alice")     // supprime ; sur une clé absente : ne fait rien, aucune erreur
n := len(m)            // nombre de paires
clear(m)               // Go 1.21+ : vide la map en gardant sa capacité
```

### L'ordre d'itération est **délibérément** aléatoire

```go
for k, v := range m {
	fmt.Println(k, v)     // ordre différent à CHAQUE exécution
}
```

Ce n'est pas un effet de bord de l'implémentation : le runtime Go **randomise
volontairement** le point de départ. La raison est pédagogique et défensive — empêcher tout
code de dépendre d'un ordre que la spécification ne garantit pas. Sans cette randomisation,
un programme marcherait par chance pendant deux ans puis casserait à la mise à jour de Go.

Pour une sortie déterministe, il faut trier les clés :

```go
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
slices.Sort(keys)             // Go 1.21+
for _, k := range keys {
	fmt.Println(k, m[k])
}

// Plus court depuis Go 1.23 (itérateurs) :
for _, k := range slices.Sorted(maps.Keys(m)) {
	fmt.Println(k, m[k])
}
```

Le second exemple utilise `maps.Keys`, qui retourne un **itérateur** (`iter.Seq`), et
`slices.Sorted`, qui le consomme en slice trié. C'est l'idiome moderne, à connaître :
beaucoup de code en ligne est antérieur.

### Contraintes sur les clés

Le type de clé doit être **comparable** avec `==` :

| Clé valide | Clé invalide |
|---|---|
| `string`, tous les numériques, `bool` | `[]T` (slice) |
| pointeurs, channels | `map[K]V` |
| tableaux `[N]T` dont les éléments sont comparables | fonctions |

```go
m := map[[2]int]string{{1, 2}: "point"}   // OK : un TABLEAU est comparable
// m2 := map[[]int]string{}               // ERREUR : invalid map key type
```

Au niveau 2 s'ajouteront les **structs** (comparables si tous leurs champs le sont) et les
**interfaces** (qui compilent toujours mais peuvent paniquer à l'exécution si la valeur
dynamique n'est pas comparable).

Une subtilité à connaître dès maintenant : `NaN` comme clé flottante est pathologique,
puisque `NaN != NaN` — la valeur devient irrécupérable. Ne jamais utiliser de flottants en
clés.

### Ce qu'une map n'est pas

- **Pas ordonnée**, ni par insertion ni par clé.
- **Pas sûre en concurrence.** Deux goroutines qui écrivent simultanément déclenchent
  `fatal error: concurrent map writes` — une erreur fatale non récupérable, pas une panique
  ordinaire. Le runtime détecte volontairement ce cas. Solutions au niveau 4 : `sync.Mutex`
  ou `sync.Map`.
- **Pas adressable** : `&m["k"]` ne compile pas, et `m["k"].Field = 1` non plus quand la
  valeur est une struct. Il faut lire, modifier, réécrire — ou stocker des pointeurs
  (`map[string]*User`).

### L'ensemble (`set`)

Go n'a pas de type ensemble. La convention est une map dont la valeur ne sert à rien :

```go
set := map[string]bool{}
set["a"] = true
if set["a"] { … }              // lisible : une clé absente donne false
delete(set, "a")
```

Une seconde forme existe, `map[T]struct{}`, qui n'occupe aucun octet par valeur. Elle utilise
le type vide `struct{}`, que l'on comprendra pleinement au niveau 2. **Recommandation :
`map[T]bool` par défaut** — la lisibilité l'emporte, et l'économie de mémoire ne compte que
sur des ensembles de plusieurs millions d'entrées.

## Exemple

Aucune fonction passée en paramètre ici : les types fonction arrivent à la leçon 11.

```go
package main

import (
	"fmt"
	"maps"
	"slices"
	"strings"
)

// WordCount compte les occurrences de chaque mot, en ignorant la casse.
func WordCount(text string) map[string]int {
	counts := make(map[string]int)
	for _, w := range strings.Fields(strings.ToLower(text)) {
		w = strings.Trim(w, ".,!?;:\"'")
		if w == "" {
			continue
		}
		counts[w]++ // lire une clé absente donne 0 : l'incrément fonctionne d'emblée
	}
	return counts
}

// MostFrequent retourne le mot le plus fréquent.
// En cas d'égalité, le plus petit dans l'ordre alphabétique gagne : la sortie
// est ainsi DÉTERMINISTE, ce que l'itération d'une map ne garantit jamais.
func MostFrequent(counts map[string]int) (string, int, bool) {
	best, bestN := "", 0
	found := false
	for _, w := range slices.Sorted(maps.Keys(counts)) { // ordre stable garanti
		if !found || counts[w] > bestN {
			best, bestN, found = w, counts[w], true
		}
	}
	return best, bestN, found
}

// Vocabulary retourne l'ensemble des mots distincts, sous forme d'ensemble.
func Vocabulary(counts map[string]int) map[string]bool {
	set := make(map[string]bool, len(counts))
	for w := range counts { // seule la clé nous intéresse
		set[w] = true
	}
	return set
}

func main() {
	text := "Le chat dort. Le chien dort aussi, et le chat rêve."
	counts := WordCount(text)

	// Sortie déterministe : on trie les clés (l'ordre d'une map est aléatoire)
	for _, w := range slices.Sorted(maps.Keys(counts)) {
		fmt.Printf("%-8s %d\n", w, counts[w])
	}

	if w, n, ok := MostFrequent(counts); ok {
		fmt.Printf("\nmot le plus fréquent : %q (%d fois)\n", w, n)
	}

	// L'idiome « virgule ok »
	if n, ok := counts["chat"]; ok {
		fmt.Printf("« chat » apparaît %d fois\n", n)
	}
	if _, ok := counts["souris"]; !ok {
		fmt.Println("« souris » est absent")
	}

	vocab := Vocabulary(counts)
	fmt.Printf("%d mots distincts ; « dort » connu : %v\n", len(vocab), vocab["dort"])
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `counts[w]++` | Fonctionne sur une clé absente : lecture → 0, incrément → 1. Idiome central du comptage. |
| `slices.Sorted(maps.Keys(counts))` | `maps.Keys` retourne un **itérateur** (Go 1.23+), `slices.Sorted` le consomme en slice trié. C'est l'idiome moderne pour parcourir une map dans un ordre stable. |
| Le parcours trié dans `MostFrequent` | Sans lui, deux mots de même fréquence donneraient un gagnant **différent à chaque exécution** : le test passerait en local et échouerait un jour en intégration continue. Point de qualité, pas de style. |
| `for w := range counts` | Avec une seule variable, `range` sur une map ne donne que **la clé**. |
| `make(map[string]bool, len(counts))` | Préallocation : évite les rehachages successifs. |
| `if _, ok := …; !ok` | Test d'absence pur : la valeur est ignorée avec `_`. |

## Erreurs fréquentes

1. **Écrire dans une map `nil`** → panique. Cause n°1 : un champ de struct non initialisé.
2. **Confondre « absent » et « zéro »** : `if m[k] == 0` n'est pas `if _, ok := m[k]; !ok`.
3. **Dépendre de l'ordre d'itération.** Le bug se manifestera en CI, pas en local.
4. **Écrire depuis plusieurs goroutines** → `fatal error: concurrent map writes`, non récupérable.
5. **`&m[k]`** ou `m[k].Field = v` → ne compile pas : les entrées d'une map ne sont pas adressables.
6. **Utiliser un slice comme clé** → erreur de compilation.
7. **Muter une map passée en paramètre** sans le documenter : une map est une **référence**, l'appelant voit les modifications.
8. **Croire que `delete` libère la mémoire** : la map conserve sa capacité. `clear(m)` non plus. Pour vraiment libérer, réaffecter une nouvelle map.

## Bonnes pratiques Go

- Toujours initialiser une map avant d'écrire : `make` ou littéral.
- `make(map[K]V, n)` quand la taille est estimable : évite les rehachages.
- L'idiome `v, ok` dès que la zéro-valeur est ambiguë.
- Pour toute sortie utilisateur, **trier les clés** : le déterminisme est un critère de qualité, pas un luxe.
- Documenter si une fonction modifie la map reçue.
- `map[T]bool` pour un ensemble ; `struct{}{}` seulement si la mémoire est un enjeu mesuré.
- Une map en champ de struct partagée entre goroutines exige une protection **dès le premier jour** — la rajouter après coup est douloureux.

## Ce que je dois retenir

- Lire une clé absente donne la **zéro-valeur** ; `v, ok := m[k]` lève l'ambiguïté.
- **Écrire dans une map `nil` panique** — la lire, non.
- L'ordre d'itération est **volontairement aléatoire** : trier les clés pour être déterministe.
- Les clés doivent être **comparables** : pas de slice, pas de map, pas de fonction.
- Une map n'est **pas sûre en concurrence** ; l'erreur est fatale, pas récupérable.
- Les entrées ne sont **pas adressables**.
- Une map se comporte comme une référence : la passer permet à la fonction de la modifier.

🧵 **Fil rouge :** cette leçon ajoute une ligne au tableau *[Copié ou partagé ?](../../ressources/copie-ou-partage.md)* — c'est le moment de le relire.

➡️ [Exercices](exercices.md)
