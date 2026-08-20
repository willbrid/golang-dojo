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
une map `nil`. C'est la première cause de panique chez les débutants, en général via un
champ de struct oublié :

```go
type Cache struct { data map[string]int }
c := Cache{}      // c.data est nil
c.data["k"] = 1   // PANIQUE
```

Toujours initialiser une map dans son constructeur.

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
| interfaces | fonctions |
| structs et tableaux **dont tous les champs sont comparables** | struct contenant un slice |

```go
type Point struct{ X, Y int }
m := map[Point]string{{1, 2}: "origine décalée"}   // OK : struct comparable

type Bad struct{ Data []byte }
m2 := map[Bad]string{}    // ERREUR de compilation : invalid map key type
```

Deux subtilités :
- Utiliser une **interface** comme clé compile toujours, mais panique à l'exécution si la
  valeur dynamique n'est pas comparable (`map[any]string` avec un slice dedans).
- `NaN` comme clé flottante est pathologique : `NaN != NaN`, donc la valeur devient
  irrécupérable. Ne jamais utiliser de flottants en clés.

### Ce qu'une map n'est pas

- **Pas ordonnée**, ni par insertion ni par clé.
- **Pas sûre en concurrence.** Deux goroutines qui écrivent simultanément déclenchent
  `fatal error: concurrent map writes` — une erreur fatale non récupérable, pas une panique
  ordinaire. Le runtime détecte volontairement ce cas. Solutions au niveau 4 : `sync.Mutex`
  ou `sync.Map`.
- **Pas adressable** : `&m["k"]` ne compile pas, et `m["k"].Field = 1` non plus quand la
  valeur est une struct. Il faut lire, modifier, réécrire — ou stocker des pointeurs
  (`map[string]*User`).

### L'ensemble (`set`) : `map[T]struct{}` ou `map[T]bool` ?

Go n'a pas de type ensemble. Deux conventions :

```go
set := map[string]struct{}{}     // 0 octet par valeur
set["a"] = struct{}{}
_, exists := set["a"]

set2 := map[string]bool{}        // 1 octet par valeur
set2["a"] = true
if set2["a"] { … }               // plus lisible
```

`struct{}` est le type vide : il n'occupe **aucun** octet. Sur un ensemble d'un million
d'entrées, l'économie est réelle mais modeste. `map[T]bool` se lit mieux et permet
`if set[k]` directement. **Recommandation : `map[T]bool` par défaut**, `struct{}{}` quand
la mémoire compte vraiment et que le code est bien commenté.

## Exemple

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

// TopN retourne les n mots les plus fréquents, ordre déterministe garanti :
// fréquence décroissante, puis alphabétique en cas d'égalité.
func TopN(counts map[string]int, n int) []string {
	words := slices.Collect(maps.Keys(counts)) // Go 1.23+ : itérateur → slice

	slices.SortFunc(words, func(a, b string) int {
		if counts[a] != counts[b] {
			return counts[b] - counts[a] // décroissant
		}
		return strings.Compare(a, b) // départage stable
	})

	return words[:min(n, len(words))] // min est intégré depuis Go 1.21
}

func main() {
	text := "Le chat dort. Le chien dort aussi, et le chat rêve."
	counts := WordCount(text)

	for _, w := range TopN(counts, 3) {
		fmt.Printf("%-8s %d\n", w, counts[w])
	}

	if n, ok := counts["chat"]; ok {
		fmt.Printf("\n« chat » apparaît %d fois\n", n)
	}
	if _, ok := counts["souris"]; !ok {
		fmt.Println("« souris » est absent")
	}
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `counts[w]++` | Fonctionne sur une clé absente : lecture → 0, incrément → 1. Idiome central du comptage. |
| `slices.Collect(maps.Keys(counts))` | `maps.Keys` retourne un **itérateur** (Go 1.23+), `slices.Collect` le matérialise. Avant : une boucle `append` manuelle. |
| `slices.SortFunc` avec départage | Sans le `strings.Compare` final, deux mots de même fréquence sortiraient dans un ordre **aléatoire** : la sortie ne serait pas reproductible. Point de qualité, pas de style. |
| `counts[b] - counts[a]` | Convention de `SortFunc` : négatif si `a` avant `b`. *(Attention : la soustraction peut déborder sur de très grands entiers ; `cmp.Compare` est plus sûr.)* |
| `min(n, len(words))` | `min` et `max` sont des fonctions **intégrées** depuis Go 1.21, sans import. |
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

➡️ [Exercices](exercices.md)
