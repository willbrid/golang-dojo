# Leçon 4 — pprof : le profil CPU

> Un benchmark dit **combien** une fonction coûte ; il ne dit pas **où** passe le temps à l'intérieur d'un
> programme. Pour cela, on échantillonne : cent fois par seconde, on note ce que chaque fil d'exécution est
> en train de faire, et on compte. C'est le **profil CPU**, et Go le fournit d'origine avec `pprof`. Cette
> leçon apprend à le collecter (dans un test, dans un programme, dans un service en production), à le lire,
> et à s'en servir pour corriger, mesurer, puis recommencer — sans jamais deviner.

## Objectifs

1. Collecter un profil CPU avec `go test -cpuprofile`, `runtime/pprof` et `net/http/pprof`.
2. Lire `go tool pprof` : `top`, `top -cum`, `list`, la vue graphique et le *flamegraph*.
3. Distinguer temps **propre** (*flat*) et temps **cumulé** (*cum*).
4. Suivre la boucle : profiler, corriger le premier poste, vérifier, reprofiler.
5. Savoir ce qu'est l'optimisation guidée par profil (PGO).

## Explication

### Collecter

Trois façons, selon le contexte :

```sh
# 1. Dans un benchmark (le plus simple) :
go test -run XXX -bench TopSlow -cpuprofile cpu.out
go tool pprof words.test cpu.out      # le binaire de test est laissé à côté
```

```go
// 2. Dans un programme : écrire le profil d'une portion d'exécution.
f, _ := os.Create("cpu.out")
pprof.StartCPUProfile(f) // paquet runtime/pprof
defer pprof.StopCPUProfile()
```

```go
// 3. Dans un service qui tourne : les routes /debug/pprof/, sur une adresse d'ADMINISTRATION.
import _ "net/http/pprof" // enregistre ses routes sur http.DefaultServeMux

go http.ListenAndServe("127.0.0.1:6060", nil) // jamais exposé à Internet (N10 L10)
```

Puis, depuis la machine de l'exploitant : `go tool pprof http://127.0.0.1:6060/debug/pprof/profile?seconds=30`
collecte 30 secondes de profil **en production**, sur le vrai trafic. Le coût de l'échantillonnage est
faible (de l'ordre de quelques pour cent pendant la collecte), ce qui permet de le faire sur un service en
service.

Le profil est un **échantillonnage** : environ 100 fois par seconde et par processeur, le *runtime* note la
pile d'appels en cours. Une fonction qui apparaît dans 40 % des échantillons a consommé environ 40 % du temps
processeur. Il faut donc assez d'échantillons (quelques secondes de travail) pour que les proportions soient
fiables.

### Lire : *flat* et *cum*

```
      flat  flat%   sum%        cum   cum%
     110ms  7.91%  7.91%      420ms 30.22%  regexp/syntax.appendRange
      90ms  6.47% 22.30%      510ms 36.69%  regexp/syntax.appendTable
```

- **flat** : le temps passé **dans** la fonction elle-même ;
- **cum** : le temps passé dans la fonction **et tout ce qu'elle appelle**.

`top` trie par *flat* : il montre les fonctions **feuilles** où le processeur travaille, souvent dans la
bibliothèque standard ou le *runtime* (`memmove`, `mallocgc`…), ce qui ne dit pas quoi changer. `top -cum`
trie par *cum* : il montre les fonctions **de votre code** qui, directement ou non, coûtent le plus. On
commence presque toujours par `top -cum`, puis on descend.

`list NomDeFonction` affiche le source avec le temps par ligne : c'est là qu'on voit la coupable.

```
         .      860ms     22:		re := regexp.MustCompile(`[\p{L}']+`) // compilée à chaque ligne
         .      140ms     23:		for _, w := range re.FindAllString(line, -1) {
         .       50ms     24:			counts[strings.ToLower(w)]++
```

La vue interactive (`go tool pprof -http=:8080 words.test cpu.out`) ajoute le graphe d'appels et le
***flamegraph*** : chaque pile est une colonne, chaque fonction un rectangle dont la **largeur** est
proportionnelle à son temps cumulé ; les rectangles larges en haut de la pile sont les cibles. (La vue
graphique demande Graphviz ; le *flamegraph* et `top` non.)

### La boucle de travail

1. **Profiler** sur une charge réaliste.
2. Prendre le **premier** poste de `top -cum` dans votre code ; `list` pour la ligne.
3. **Corriger** cela seulement.
4. **Vérifier** : même résultat (un test d'équivalence), et un benchmark avant/après (leçon 1).
5. **Reprofiler** : le paysage a changé, le poste suivant n'est plus le même.

C'est exactement le déroulement de l'exemple :

| Version | Temps (médiane de 5) | Mémoire / op | Allocations / op | Premier poste du profil |
|---|---|---|---|---|
| `TopSlow` | 570 ms | 459 Mo | 780 974 | `regexp.MustCompile` (62 % du temps), dans la boucle |
| `TopRegexp` (compilée une fois) | 191 ms | 27 Mo | 240 098 | `FindAllString` (76 %) |
| `TopScan` (découpage manuel) | 36 ms | 4 Mo | 240 030 | `strings.ToLower` (61 %) |

Trois profils, trois cibles différentes, un facteur 16 au total. Deviner aurait mené ailleurs : ni le tri
final ni la map, qu'on soupçonne volontiers, n'apparaissent en tête.

### L'optimisation guidée par profil (PGO)

Le compilateur Go sait utiliser un profil CPU pour mieux optimiser (*inlining* plus agressif des fonctions
chaudes, notamment) : il suffit de placer un profil représentatif de la production dans le paquet
principal sous le nom `default.pgo`, et `go build` l'utilise automatiquement (Go 1.21). Les gains annoncés
par l'équipe Go sont de l'ordre de 2 à 14 % ; ils dépendent du programme et se mesurent (exercice E5). C'est
un gain **gratuit** une fois le profil collecté, mais pas un substitut à la correction d'un algorithme
coûteux.

### Quand ne pas profiler… et ce qu'il ne montre pas

- Le profil CPU ne montre que le temps **processeur**. Une requête lente parce qu'elle **attend** (la base,
  le réseau, un verrou) n'y apparaît pas : il faut le profil de blocage (leçon 5) ou une trace (N10 L10).
- Sur une charge irréaliste (dix entrées au lieu de dix mille), le profil désigne de faux coupables.
- Pas besoin de profil pour un programme dont la performance est suffisante.

## Exemple

Un compteur des mots les plus fréquents, en trois versions : naïve, puis corrigée deux fois en suivant le
profil. Un test vérifie que les trois rendent le même résultat.

```go
// Fichier : words.go
// Package words : les mots les plus fréquents d'un texte.
package words

import (
	"cmp"
	"regexp"
	"slices"
	"sort"
	"strings"
	"unicode"
)

type Count struct {
	Word string
	N    int
}

// TopSlow : la version « qui marche ». Où passe le temps ? Le profil le dira.
func TopSlow(text string, k int) []Count {
	counts := map[string]int{}
	for _, line := range strings.Split(text, "\n") {
		re := regexp.MustCompile(`[\p{L}']+`) // compilée à chaque ligne
		for _, w := range re.FindAllString(line, -1) {
			counts[strings.ToLower(w)]++
		}
	}
	var out []Count
	for w, n := range counts {
		out = append(out, Count{w, n})
	}
	sort.Slice(out, func(i, j int) bool {
		if out[i].N != out[j].N {
			return out[i].N > out[j].N
		}
		return out[i].Word < out[j].Word
	})
	if len(out) > k {
		out = out[:k]
	}
	return out
}

var wordRe = regexp.MustCompile(`[\p{L}']+`)

// TopRegexp : étape 1, l'expression régulière compilée UNE fois.
func TopRegexp(text string, k int) []Count {
	counts := map[string]int{}
	for _, w := range wordRe.FindAllString(text, -1) {
		counts[strings.ToLower(w)]++
	}
	return top(counts, k)
}

// TopScan : étape 2, un découpage à la main, sans expression régulière.
func TopScan(text string, k int) []Count {
	counts := map[string]int{}
	start := -1
	for i, r := range text {
		if unicode.IsLetter(r) || r == '\'' {
			if start < 0 {
				start = i
			}
			continue
		}
		if start >= 0 {
			counts[strings.ToLower(text[start:i])]++
			start = -1
		}
	}
	if start >= 0 {
		counts[strings.ToLower(text[start:])]++
	}
	return top(counts, k)
}

func top(counts map[string]int, k int) []Count {
	out := make([]Count, 0, len(counts))
	for w, n := range counts {
		out = append(out, Count{w, n})
	}
	slices.SortFunc(out, func(a, b Count) int {
		if c := cmp.Compare(b.N, a.N); c != 0 {
			return c
		}
		return strings.Compare(a.Word, b.Word)
	})
	return out[:min(k, len(out))]
}
```

```go
// Fichier : words_test.go
package words

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
)

// Un texte de 20 000 lignes, vocabulaire de 5 000 mots, fréquences inégales.
var text = func() string {
	rng := rand.New(rand.NewPCG(1, 2))
	vocab := make([]string, 5000)
	for i := range vocab {
		// des mots faits de lettres seulement : les chiffres de i deviennent a…j
		vocab[i] = "Mot" + strings.Map(func(r rune) rune { return 'a' + r - '0' }, fmt.Sprint(i)) + "é"
	}
	var b strings.Builder
	for range 20_000 {
		for j := range 12 {
			if j > 0 {
				b.WriteByte(' ')
			}
			i := int(rng.ExpFloat64()*400) % len(vocab)
			b.WriteString(vocab[i])
		}
		b.WriteByte('\n')
	}
	return b.String()
}()

func BenchmarkTopSlow(b *testing.B) {
	for b.Loop() {
		TopSlow(text, 10)
	}
}

func BenchmarkTopRegexp(b *testing.B) {
	for b.Loop() {
		TopRegexp(text, 10)
	}
}

func BenchmarkTopScan(b *testing.B) {
	for b.Loop() {
		TopScan(text, 10)
	}
}

func TestSameResult(t *testing.T) {
	want := TopSlow(text, 10)
	for name, f := range map[string]func(string, int) []Count{"regexp": TopRegexp, "scan": TopScan} {
		if got := f(text, 10); !slices.Equal(got, want) {
			t.Errorf("%s : %v\nattendu %v", name, got, want)
		}
	}
	t.Log(want[:3])
}
```

```sh
go test -run XXX -bench . -benchmem -count 5
go test -run XXX -bench TopSlow -cpuprofile cpu.out
go tool pprof -top -cum words.test cpu.out
go tool pprof -list 'words.TopSlow' words.test cpu.out
```

Les mesures sont dans le tableau de l'explication. `go test -run Same -v` affiche les trois mots les plus
fréquents, identiques pour les trois versions : `[{motcbé 638} {mothé 612} {motbgé 610}]`.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `text` construit une fois, de façon déterministe | Une charge réaliste (20 000 lignes, 5 000 mots, fréquences inégales) et identique à chaque exécution. |
| Vocabulaire fait de **lettres** seulement | Une première version avec des chiffres (`Mot12é`) se découpait en deux « mots » : les mesures portaient sur un cas dégénéré. Le test d'équivalence l'a révélé. |
| `TestSameResult` | Une optimisation qui change le résultat est un bug : on vérifie avant de mesurer. |
| `regexp.MustCompile` dans la boucle | 62 % du temps : compiler une expression régulière coûte bien plus que l'appliquer. |
| `wordRe` au niveau du paquet | Compilée une fois, au démarrage ; `*regexp.Regexp` est sûr en concurrence. |
| `TopScan` avec `unicode.IsLetter` | Un parcours unique du texte, sans machine à états d'expression régulière. |
| `text[start:i]` | Une sous-chaîne ne copie rien ; seul `strings.ToLower` alloue — la cible suivante. |
| `top` et `slices.SortFunc` + `cmp.Compare` | Tri par fréquence décroissante puis par mot : un ordre total, donc des résultats comparables. |

## Erreurs fréquentes

1. **Optimiser sans profil**, au flair.
2. **Lire `top` (flat) et conclure** qu'il faut optimiser `memmove` ou `mallocgc`.
3. **Corriger plusieurs choses à la fois** : impossible de savoir laquelle a compté.
4. **Ne pas reprofiler** après une correction.
5. **Profiler une charge irréaliste.**
6. **Exposer `/debug/pprof` sur l'adresse publique** du service.
7. **Chercher dans le profil CPU** une lenteur qui vient d'une attente.
8. **Changer le résultat en optimisant**, faute de test d'équivalence.

## Bonnes pratiques Go

- `-cpuprofile` sur un benchmark réaliste ; `net/http/pprof` sur une adresse d'administration en production.
- `top -cum` d'abord, `list` ensuite, *flamegraph* pour la vue d'ensemble.
- Une correction à la fois ; test d'équivalence ; benchmark avant/après ; nouveau profil.
- Expressions régulières compilées une fois ; découpage manuel sur les chemins très chauds.
- Un profil de production en `default.pgo` pour profiter de PGO.

## Ce que je dois retenir

- Le profil CPU échantillonne les piles ~100 fois par seconde ; il dit où passe le temps processeur.
- *flat* : dans la fonction ; *cum* : avec tout ce qu'elle appelle. Commencer par `top -cum`.
- `list` montre la ligne coupable.
- Profiler, corriger le premier poste, vérifier, reprofiler : la cible change à chaque tour.
- Le profil CPU ne voit pas les attentes ; PGO réutilise un profil pour mieux compiler.

---

➡️ [Exercices](exercices.md)
