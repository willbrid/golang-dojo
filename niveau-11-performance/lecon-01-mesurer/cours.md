# Leçon 1 — Mesurer

> « Cette version est plus rapide » est une affirmation sur des **mesures**, et les mesures d'un
> programme sont bruitées : la même fonction, mesurée deux fois sur le même portable, a varié ici de
> 40 %. Une optimisation de 10 % « constatée » sur une seule exécution peut n'être que du bruit — ou
> masquer une régression. Cette leçon apprend à écrire des benchmarks qui mesurent ce qu'on croit
> mesurer, à quantifier le bruit, et à **comparer** deux versions avec un test statistique, comme le fait
> l'outil officiel `benchstat`.

## Objectifs

1. Écrire un benchmark correct avec `b.Loop`, et savoir ce qu'il mesure.
2. Lire `ns/op`, `B/op`, `allocs/op`.
3. Connaître les sources de bruit et les réduire.
4. Répéter les mesures (`-count`) et les résumer par la médiane et l'étendue.
5. Comparer deux séries par un test de rang, et conclure « pas de différence démontrée » quand c'est le cas.
6. Distinguer micro-benchmark et mesure de bout en bout.

## Explication

### Un benchmark correct

```go
func BenchmarkJoin(b *testing.B) {
	for b.Loop() {
		Join(parts)
	}
}
```

`b.Loop` (Go 1.24, N5 L6) fait tourner la boucle assez longtemps pour une mesure stable, n'inclut dans le
chronomètre que ce qui est **dans** la boucle (la préparation au-dessus est exclue), et empêche le
compilateur d'éliminer l'appel dont le résultat n'est pas utilisé. Les anciennes formes
(`for i := 0; i < b.N; i++`) restent valides mais demandent ces précautions à la main (`b.ResetTimer`,
affecter le résultat à une variable globale).

Ce qu'on mesure doit **ressembler** au cas réel : des données de taille et de forme réalistes, préparées hors
de la boucle, et le code écrit comme il le sera en production (N9 L4 a montré une boucle deux fois plus lente
parce qu'elle portait sur une variable globale dans le corps du benchmark).

### Lire une ligne

```
BenchmarkConcat-8    26470    46421 ns/op    135328 B/op    198 allocs/op
```

- `-8` : la valeur de `GOMAXPROCS` (le nombre de processeurs utilisables) ;
- `26470` : le nombre d'itérations exécutées ;
- `ns/op` : le temps moyen par itération ;
- `B/op` et `allocs/op` (avec `-benchmem`) : la mémoire allouée **sur le tas** et le nombre d'allocations
  par itération. Ce sont souvent les chiffres les plus parlants, et ils sont **déterministes** : ils ne
  varient pas d'une exécution à l'autre, contrairement au temps.

Pour joindre 100 chaînes (mesuré) :

| Version | Temps (médiane) | Mémoire | Allocations |
|---|---|---|---|
| `s += p` | 45 840 ns | 135 328 o | 198 |
| `strings.Builder` | 1 942 ns | 3 312 o | 8 |
| `strings.Join` | 1 310 ns | 1 408 o | 1 |

`+=` recopie toute la chaîne à chaque tour (une nouvelle chaîne de plus en plus longue) ; `Builder` agrandit
un tampon en doublant sa capacité (8 allocations) ; `Join` calcule d'abord la taille totale et alloue
**une** fois.

### Le bruit

La même mesure de `Join`, dix fois de suite, a varié de 1 167 à 1 640 ns (±18 % autour de la médiane). Les
causes, sur une machine ordinaire :

- la **fréquence du processeur** varie (turbo, économie d'énergie, échauffement) — sur un portable, c'est
  souvent la première cause ;
- les **autres programmes** (navigateur, éditeur, indexation) et les interruptions du système ;
- le **ramasse-miettes** et l'ordonnanceur de Go eux-mêmes ;
- l'**état des caches** et l'alignement du code (deux binaires presque identiques peuvent différer de
  quelques pour cent pour une simple question de placement).

Réduire le bruit : machine au repos, sur secteur ; plusieurs exécutions (`-count 10`) ; une durée par mesure
suffisante (`-benchtime`) ; et, pour comparer, alterner ou répéter les séries plutôt que de mesurer A le
matin et B l'après-midi. On ne **supprime** pas le bruit : on le **quantifie**, et on en tient compte.

### Comparer : la médiane et un test de rang

Une moyenne est déformée par les valeurs extrêmes (une exécution perturbée par un processus de fond), et
les temps d'exécution ne suivent pas une loi normale : ils ont une « queue » vers le haut. D'où deux
choix, qui sont ceux de `benchstat` (l'outil officiel de l'équipe Go, `golang.org/x/perf/cmd/benchstat`) :

- résumer chaque série par sa **médiane** et son étendue ;
- comparer deux séries par un **test de Mann-Whitney** : il range toutes les mesures ensemble et regarde si
  celles d'une série sont systématiquement aux premiers rangs. La **p-valeur** est la probabilité
  d'observer un tel déséquilibre si les deux versions étaient en réalité identiques. Sous 0,05, on retient la
  différence ; au-dessus, on conclut « **pas de différence démontrée** » — ce qui n'est pas « pas de
  différence », mais « le bruit pourrait suffire à l'expliquer ».

L'exemple implémente ce calcul (quelques dizaines de lignes) et l'applique aux mesures ci-dessus. Le cas
le plus instructif est le premier : le **même** code mesuré deux fois donne des médianes de 1 310 et 1 277
ns, soit un « gain » de 2,5 % qui n'existe pas (p = 0,36). Sans le test, on aurait pu publier cette
« optimisation ».

En pratique : `go test -bench . -count 10 > avant.txt`, modification, `go test -bench . -count 10 >
apres.txt`, puis `benchstat avant.txt apres.txt` (installé par `go install
golang.org/x/perf/cmd/benchstat@latest`). Il affiche la médiane, l'intervalle de confiance, la variation
et la p-valeur, ou `~` quand la différence n'est pas significative.

### Micro et macro

Un micro-benchmark mesure une fonction isolée, dans des conditions idéales : données chaudes en cache, rien
d'autre ne tourne. Il est indispensable pour comparer deux implémentations, mais il ne dit pas si cette
fonction **compte** dans le service : un gain de 30 % sur une fonction qui représente 0,1 % du temps d'une
requête ne se verra jamais. Il faut aussi des mesures **de bout en bout** (un test de charge sur le service,
leçon 7) et des **profils** (leçons 4 et 5) pour savoir où le temps passe réellement.

## Exemple

Trois façons de joindre des chaînes, leurs benchmarks, puis un comparateur qui résume et compare les
séries mesurées (recopiées dans le programme pour que sa sortie soit reproductible).

```go
// Fichier : join.go
package join

import "strings"

// Concat : la version naïve, une nouvelle chaîne à chaque tour.
func Concat(parts []string) string {
	s := ""
	for i, p := range parts {
		if i > 0 {
			s += ","
		}
		s += p
	}
	return s
}

// Builder : un seul tampon, agrandi au besoin.
func Builder(parts []string) string {
	var b strings.Builder
	for i, p := range parts {
		if i > 0 {
			b.WriteByte(',')
		}
		b.WriteString(p)
	}
	return b.String()
}

// Join : la bibliothèque standard, qui calcule la taille totale d'abord.
func Join(parts []string) string { return strings.Join(parts, ",") }
```

```go
// Fichier : join_test.go
package join

import (
	"fmt"
	"testing"
)

var parts = func() []string {
	p := make([]string, 100)
	for i := range p {
		p[i] = fmt.Sprintf("élément-%d", i)
	}
	return p
}()

func BenchmarkConcat(b *testing.B) {
	for b.Loop() {
		Concat(parts)
	}
}

func BenchmarkBuilder(b *testing.B) {
	for b.Loop() {
		Builder(parts)
	}
}

func BenchmarkJoin(b *testing.B) {
	for b.Loop() {
		Join(parts)
	}
}
```

`go test -bench . -benchmem -count 10` produit dix lignes par benchmark ; les valeurs `ns/op` de deux telles
séries sont reprises ci-dessous.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"slices"
)

// Les mesures de BenchmarkJoin, BenchmarkBuilder et BenchmarkConcat (ns/op), deux séries
// de 10 exécutions, recopiées telles quelles depuis la sortie de go test -count 10.
var (
	join1   = []float64{1176, 1167, 1203, 1640, 1308, 1204, 1534, 1464, 1531, 1311}
	join2   = []float64{1292, 1262, 1309, 1603, 1324, 1304, 1183, 1167, 1195, 1171}
	builder = []float64{2148, 2081, 1929, 1922, 2682, 2160, 1903, 1827, 1955, 1837}
	concat  = []float64{46421, 47018, 49277, 44390, 44682, 45266, 46240, 45662, 44757, 46019}
)

func median(xs []float64) float64 {
	s := slices.Sorted(slices.Values(xs))
	n := len(s)
	if n%2 == 1 {
		return s[n/2]
	}
	return (s[n/2-1] + s[n/2]) / 2
}

// mannWhitney rend la p-valeur bilatérale du test de Mann-Whitney (approximation normale) :
// la probabilité d'observer un tel écart de RANGS si les deux séries venaient de la même loi.
// Il ne suppose pas que les mesures suivent une loi normale — ce qu'elles ne font pas.
func mannWhitney(a, b []float64) float64 {
	type obs struct {
		v float64
		a bool
	}
	var all []obs
	for _, v := range a {
		all = append(all, obs{v, true})
	}
	for _, v := range b {
		all = append(all, obs{v, false})
	}
	slices.SortFunc(all, func(x, y obs) int { return cmp.Compare(x.v, y.v) })
	var ra float64 // somme des rangs de a (rang moyen en cas d'égalité)
	for i := 0; i < len(all); {
		j := i
		for j < len(all) && all[j].v == all[i].v {
			j++
		}
		rank := float64(i+j+1) / 2
		for k := i; k < j; k++ {
			if all[k].a {
				ra += rank
			}
		}
		i = j
	}
	n1, n2 := float64(len(a)), float64(len(b))
	u := ra - n1*(n1+1)/2
	mean, sd := n1*n2/2, math.Sqrt(n1*n2*(n1+n2+1)/12)
	z := (math.Abs(u-mean) - 0.5) / sd // correction de continuité
	return math.Erfc(z / math.Sqrt2)
}

func compare(name string, a, b []float64) {
	ma, mb := median(a), median(b)
	p := mannWhitney(a, b)
	verdict := fmt.Sprintf("%+.1f %%", (mb-ma)/ma*100)
	if p > 0.05 {
		verdict = "~ (pas de différence démontrée)"
	}
	fmt.Printf("%-22s %8.0f ns → %8.0f ns   p=%.3f   %s\n", name, ma, mb, p, verdict)
}

func main() {
	for _, s := range []struct {
		name string
		xs   []float64
	}{{"Join (série 1)", join1}, {"Join (série 2)", join2}, {"Builder", builder}, {"Concat", concat}} {
		fmt.Printf("%-15s médiane %7.0f ns   étendue %6.0f – %6.0f (±%2.0f %%)\n", s.name, median(s.xs),
			slices.Min(s.xs), slices.Max(s.xs), (slices.Max(s.xs)-slices.Min(s.xs))/2/median(s.xs)*100)
	}
	fmt.Println()
	compare("Join, série 1 → 2", join1, join2) // le MÊME code mesuré deux fois
	compare("Builder → Join", builder, join1)
	compare("Concat → Builder", concat, builder)
}
```

Sortie :

```
Join (série 1)  médiane    1310 ns   étendue   1167 –   1640 (±18 %)
Join (série 2)  médiane    1277 ns   étendue   1167 –   1603 (±17 %)
Builder         médiane    1942 ns   étendue   1827 –   2682 (±22 %)
Concat          médiane   45840 ns   étendue  44390 –  49277 (± 5 %)

Join, série 1 → 2          1310 ns →     1277 ns   p=0.364   ~ (pas de différence démontrée)
Builder → Join             1942 ns →     1310 ns   p=0.000   -32.6 %
Concat → Builder          45840 ns →     1942 ns   p=0.000   -95.8 %
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `parts` préparé hors des benchmarks | La préparation n'est pas mesurée. |
| `for b.Loop()` | Mesure, répétition et protection contre l'élimination de code. |
| Trois versions de la même fonction | Même résultat, coûts très différents : `B/op` et `allocs/op` l'expliquent. |
| `median` | Un résumé robuste aux valeurs extrêmes. |
| Étendue (±%) | Le bruit, chiffré : ±17 à ±22 % sur ce portable. |
| `mannWhitney` | Rangs, rang moyen en cas d'égalité, approximation normale avec correction de continuité ; `math.Erfc` donne la p-valeur bilatérale. |
| `p > 0.05` → `~` | La conclusion honnête : pas de différence démontrée. |
| « Join, série 1 → 2 » | Le même code : un écart apparent de 2,5 %, statistiquement nul. |
| « Concat → Builder » | −95,8 %, p ≈ 0 : une vraie différence, qu'aucun bruit n'explique. |

## Erreurs fréquentes

1. **Conclure sur une seule exécution.**
2. **Comparer des moyennes** sans regarder la dispersion.
3. **Mesurer la préparation** avec le code mesuré.
4. **Un benchmark que le compilateur peut éliminer** (résultat inutilisé, sans `b.Loop`).
5. **Des données irréalistes** (10 éléments quand la production en a 10 000, ou l'inverse).
6. **Mesurer A et B à des moments différents** sur une machine dont la charge change.
7. **Ignorer `allocs/op`**, souvent l'explication du temps.
8. **Optimiser une fonction sans savoir si elle compte** dans le temps total.

## Bonnes pratiques Go

- `b.Loop`, `-benchmem`, `-count 10` au minimum, machine au repos.
- Médiane et étendue ; comparaison par `benchstat` (ou un test de rang) ; conclure `~` sans gêne.
- Benchmarks avec des données réalistes, versionnés avec le code, relancés avant toute affirmation.
- `B/op` et `allocs/op` comme indicateurs stables ; le temps comme confirmation.
- Profils et tests de charge pour savoir quoi mesurer.

## Ce que je dois retenir

- Une mesure est bruitée : ici ±20 %. On la répète et on la résume par la médiane.
- `B/op` et `allocs/op` sont déterministes et expliquent souvent `ns/op`.
- On compare par un test statistique ; p > 0,05 = pas de différence démontrée.
- `benchstat` est l'outil standard pour cela.
- Un micro-benchmark dit **combien** ; un profil dit **où** ; un test de charge dit **si ça compte**.

---

➡️ [Exercices](exercices.md)
