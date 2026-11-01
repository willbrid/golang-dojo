# Leçon 6 — Benchmarks

> « C'est plus rapide » est une affirmation qui se **mesure**. Le paquet `testing` contient un
> outil de mesure de performance aussi intégré que les tests : les benchmarks. Cette leçon
> apprend à en écrire de fiables avec `b.Loop` (Go 1.24), à lire leur sortie, et surtout à se
> méfier de ses propres mesures. Le niveau 11 ira plus loin (profilage) ; ici on pose la règle
> qui gouvernera toute la suite : **mesurer avant d'optimiser**.

## Objectifs

1. Écrire un benchmark avec `b.Loop` et le lancer.
2. Lire `ns/op`, `B/op`, `allocs/op`.
3. Éviter les pièges classiques : préparation mesurée, résultat éliminé par le compilateur, bruit.
4. Comparer deux versions avec plusieurs mesures et `benchstat`.

## Explication

### Un benchmark

```go
func BenchmarkSlug(b *testing.B) {
	for b.Loop() {
		Slug("Ça, c'est l'été !")
	}
}
```

- Dans un `_test.go`, une fonction `BenchmarkXxx(b *testing.B)`.
- Le corps à mesurer est dans une boucle `for b.Loop() { … }` : l'outil décide **combien de
  fois** l'exécuter pour obtenir une mesure stable (environ une seconde par défaut).
- `go test` ne lance **pas** les benchmarks par défaut. Il faut `-bench` :

```bash
go test -run '^$' -bench . -benchmem ./...
```

`-run '^$'` désactive les tests (aucun nom ne correspond à « vide ») ; `-bench .` lance tous les
benchmarks ; `-benchmem` ajoute les allocations.

### `b.Loop` (Go 1.24) et l'ancienne forme `b.N`

Avant Go 1.24, on écrivait :

```go
for i := 0; i < b.N; i++ {
	Slug("…")
}
```

Cette forme a deux pièges que `b.Loop` supprime :

1. **La préparation était mesurée** si on oubliait `b.ResetTimer()` après elle. Avec `b.Loop`,
   tout ce qui précède la boucle est exclu automatiquement : la fonction de benchmark n'est
   exécutée **qu'une fois**, et seul le corps de la boucle est répété.
2. **Le compilateur pouvait supprimer l'appel** : si le résultat n'est pas utilisé et que la
   fonction n'a pas d'effet de bord, l'optimiseur a le droit de ne rien calculer, et l'on mesure
   une boucle vide (« 0.3 ns/op » est le signe classique). On contournait avec une variable
   globale « puits ». `b.Loop` empêche cette élimination pour les appels de son corps.

Tout le code neuf utilise `b.Loop`. On rencontre encore `b.N` partout.

### Lire la sortie

```
BenchmarkJoin/builder/n=1000-8   197860   6041 ns/op   6912 B/op   1 allocs/op
```

| Champ | Sens |
|---|---|
| `BenchmarkJoin/builder/n=1000` | nom, avec sous-benchmarks (`b.Run`, comme `t.Run`) |
| `-8` | `GOMAXPROCS` pendant la mesure |
| `197860` | nombre d'itérations exécutées |
| `6041 ns/op` | temps moyen par itération |
| `6912 B/op` | octets alloués sur le tas par itération |
| `1 allocs/op` | nombre d'allocations par itération |

Les deux dernières colonnes sont souvent **plus parlantes** que le temps : elles sont
déterministes (pas de bruit), et chaque allocation coûte du temps d'allocation **et** du travail
pour le ramasse-miettes (niveau 11). Réduire `allocs/op` est l'optimisation la plus fréquente.

### Le bruit

Mesurer **cinq fois** le même code, sans rien changer, sur la machine de rédaction :

```
BenchmarkJoin/builder/n=1000-8   201925   5944 ns/op
BenchmarkJoin/builder/n=1000-8   206730   5759 ns/op
BenchmarkJoin/builder/n=1000-8   208368   6824 ns/op
BenchmarkJoin/builder/n=1000-8   165872   6670 ns/op
BenchmarkJoin/builder/n=1000-8   178180   6194 ns/op
```

De 5 759 à 6 824 ns : **18 % d'écart** sans aucune modification. Une « amélioration de 10 % »
mesurée une seule fois ne veut donc **rien dire**. Les causes : autres programmes, fréquence
variable du processeur (turbo, économie d'énergie, échauffement), placement en mémoire, cache.

Pour réduire le bruit : machine au repos, sur secteur, et **plusieurs mesures** (`-count=10`).

### Comparer : `benchstat`

`benchstat` (module `golang.org/x/perf`) est l'outil officiel de comparaison :

```bash
go install golang.org/x/perf/cmd/benchstat@latest

go test -run '^$' -bench . -benchmem -count=10 ./join > avant.txt
# … modifier le code …
go test -run '^$' -bench . -benchmem -count=10 ./join > apres.txt
benchstat avant.txt apres.txt
```

Il affiche, pour chaque benchmark et chaque métrique, la **médiane** avant et après, l'écart
autour de la médiane (`± x %`), la variation (`-37.5 %`), et la **probabilité** que la
différence soit due au hasard (`p=0.000 n=10`). Si la différence n'est pas statistiquement
significative, il affiche `~` au lieu d'un pourcentage : **on ne conclut rien**.

C'est la seule façon honnête de dire « c'est plus rapide ». Dans un projet open source, une
*pull request* d'optimisation sans sortie de `benchstat` est en général renvoyée.

### Les sous-benchmarks et les tailles

Une performance dépend de la **taille de l'entrée**. Un benchmark utile mesure plusieurs
tailles, avec `b.Run` :

```go
for _, n := range []int{10, 1000} {
	b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) { … })
}
```

Ce qui est 2× plus lent à n = 10 peut être 200× plus rapide à n = 1 000 : c'est la différence
entre un coût constant et une complexité (l'exemple ci-dessous le montre).

### Ce qu'un micro-benchmark ne dit pas

Un benchmark mesure du code **isolé**, avec des caches chauds et des données identiques à
chaque tour. Le gain mesuré peut disparaître dans le programme réel (où la fonction représente
0,1 % du temps). D'où l'ordre de travail du niveau 11 : **profiler** le programme réel pour
trouver ce qui compte, **puis** écrire un benchmark de cette partie, **puis** optimiser.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : join/join.go
// Package join assemble des mots de trois façons, pour les comparer.
package join

import "strings"

// Plus concatène avec +=.
func Plus(words []string) string {
	s := ""
	for i, w := range words {
		if i > 0 {
			s += ","
		}
		s += w
	}
	return s
}

// Builder utilise un strings.Builder préalloué.
func Builder(words []string) string {
	n := len(words)
	for _, w := range words {
		n += len(w)
	}
	var b strings.Builder
	b.Grow(n)
	for i, w := range words {
		if i > 0 {
			b.WriteByte(',')
		}
		b.WriteString(w)
	}
	return b.String()
}

// Join délègue à la bibliothèque standard.
func Join(words []string) string { return strings.Join(words, ",") }
```

```go
// Fichier : join/join_test.go
package join

import (
	"fmt"
	"testing"
)

func words(n int) []string {
	w := make([]string, n)
	for i := range w {
		w[i] = fmt.Sprintf("mot%d", i)
	}
	return w
}

// TestSame : avant de comparer des vitesses, vérifier que les trois font la même chose.
func TestSame(t *testing.T) {
	in := words(50)
	if Plus(in) != Join(in) || Builder(in) != Join(in) {
		t.Fatal("les trois implémentations divergent")
	}
}

func BenchmarkJoin(b *testing.B) {
	for _, n := range []int{10, 1000} {
		in := words(n) // préparation : hors de toute mesure
		for _, impl := range []struct {
			name string
			f    func([]string) string
		}{{"plus", Plus}, {"builder", Builder}, {"join", Join}} {
			b.Run(fmt.Sprintf("%s/n=%d", impl.name, n), func(b *testing.B) {
				for b.Loop() {
					impl.f(in)
				}
			})
		}
	}
}
```

Sortie de `go test -run '^$' -bench . -benchmem ./join` sur la machine de rédaction
(Intel i5-8265U, 8 cœurs logiques) :

```
BenchmarkJoin/plus/n=10-8         	 1922366	       620.6 ns/op	     584 B/op	      18 allocs/op
BenchmarkJoin/builder/n=10-8      	15268082	        84.26 ns/op	      64 B/op	       1 allocs/op
BenchmarkJoin/join/n=10-8         	11524382	       106.0 ns/op	      64 B/op	       1 allocs/op
BenchmarkJoin/plus/n=1000-8       	    1141	   1176745 ns/op	 7187041 B/op	    1998 allocs/op
BenchmarkJoin/builder/n=1000-8    	  197860	      6041 ns/op	    6912 B/op	       1 allocs/op
BenchmarkJoin/join/n=1000-8       	  142346	     10744 ns/op	    6912 B/op	       1 allocs/op
```

Et en retirant le `b.Grow(n)` de `Builder` (cinq mesures, n = 1 000) : environ **9 500 ns/op,
24 824 B/op, 14 allocs/op** au lieu de ~6 000 ns, 6 912 B et 1 allocation.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `TestSame` | Un benchmark d'une fonction **fausse** ne sert à rien : on vérifie d'abord l'équivalence. |
| `in := words(n)` avant `b.Run` | La préparation n'est jamais mesurée. Avec `b.Loop`, même placée dans la fonction du sous-benchmark mais avant la boucle, elle ne le serait pas non plus. |
| `b.Run("plus/n=10")` | Nom hiérarchique : `-bench 'Join/builder'` filtre une implémentation. |
| `plus` à n = 1 000 | 7 Mo alloués et 1 998 allocations **par appel** : la signature d'un coût quadratique (N4 L1). 200× plus lent que `builder`. |
| `builder` contre `join` | `builder` gagne ici… parce qu'il est **spécialisé** (séparateur d'un octet). L'écart (6 µs contre 11 µs) est réel, mais le gain ne justifierait presque jamais de remplacer `strings.Join`, plus lisible. |
| Sans `Grow` | 14 allocations : le buffer double 13 fois. Le `Grow` est l'optimisation **mesurée** qui compte ici. |

## Erreurs fréquentes

1. **Une mesure unique** présentée comme une preuve.
2. **Mesurer la préparation** (avec `b.N` sans `ResetTimer`).
3. **Résultat inutilisé éliminé par le compilateur** (forme `b.N`) : des `0.3 ns/op` suspects.
4. **Benchmarker du code faux** : pas de test d'équivalence.
5. **Une seule taille d'entrée.**
6. **Comparer des mesures prises sur des machines ou à des moments différents.**
7. **Optimiser sans profil** : gagner 50 % sur 0,1 % du temps total.
8. **Ignorer `allocs/op`**, qui est souvent la vraie explication.

## Bonnes pratiques Go

- `for b.Loop()` dans tout benchmark neuf.
- `-benchmem` toujours ; `-count=10` pour comparer ; `benchstat` pour conclure.
- Plusieurs tailles d'entrée, en sous-benchmarks.
- Un test d'équivalence à côté de chaque benchmark comparatif.
- Machine au repos, mêmes conditions avant et après.
- Garder le code simple tant qu'une mesure ne prouve pas le contraire.

## Ce que je dois retenir

- `func BenchmarkXxx(b *testing.B) { for b.Loop() { … } }`.
- `go test -run '^$' -bench . -benchmem`.
- `ns/op`, `B/op`, `allocs/op` — les allocations sont déterministes.
- Le bruit dépasse facilement 10 % : `-count=10` et `benchstat`.
- `~` dans `benchstat` = aucune différence démontrée.
- Mesurer, **puis** optimiser — jamais l'inverse.

---

➡️ [Exercices](exercices.md)
