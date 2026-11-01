# Leçon 1 — Le modèle de concurrence

> Lancer une tâche en parallèle en Go tient en un mot : `go`. Cette facilité est un piège :
> elle donne l'impression que la concurrence est simple, alors que **raisonner** sur un
> programme concurrent est l'une des choses les plus difficiles du métier. Cette leçon pose le
> modèle — ce qu'est une goroutine, comment le runtime les exécute, et ce que « en même temps »
> veut dire — avant toute communication entre goroutines.

## Objectifs

1. Lancer des goroutines et comprendre leur cycle de vie.
2. Comprendre l'ordonnanceur M:N et le rôle de `GOMAXPROCS`.
3. Distinguer **concurrence** et **parallélisme**, et le mesurer.
4. Connaître les deux premiers pièges : `main` qui n'attend pas, et la closure qui capture.

## Explication

### Une goroutine

```go
go f(x)               // lance f(x) dans une nouvelle goroutine, et continue immédiatement
go func() { … }()     // idem avec une closure
```

Une **goroutine** est une fonction qui s'exécute indépendamment du reste du programme. Le mot
`go` ne bloque pas : l'appelant continue sans attendre, et sans recevoir de résultat (il n'y en a
pas : `x := go f()` n'existe pas). Communiquer un résultat demande un mécanisme — channels (leçons
2 à 4) ou mémoire partagée protégée (leçons 5 à 7).

Les arguments sont **évalués au moment du `go`**, dans la goroutine appelante, comme pour `defer`
(N2 L7) ; le corps s'exécute plus tard.

### Ce qu'une goroutine coûte

Une goroutine n'est **pas** un thread du système. Elle démarre avec une pile de quelques
kilo-octets (2 Kio aujourd'hui), qui **grandit et rétrécit** à la demande. Créer une goroutine
coûte de l'ordre de la microseconde. On peut en avoir des **centaines de milliers** dans un même
processus — un serveur Go lance typiquement une goroutine par connexion.

Un thread du système, lui, réserve une pile de l'ordre du mégaoctet et son changement de contexte
passe par le noyau. C'est cette différence qui rend le style « une goroutine par tâche » viable.

### L'ordonnanceur M:N

Le runtime de Go exécute **M** goroutines sur **N** threads du système. Trois notions :

| Lettre | Nom | Rôle |
|---|---|---|
| **G** | goroutine | le travail à faire |
| **M** | *machine* : un thread du système | ce qui exécute réellement du code |
| **P** | *processor* : un contexte d'exécution | le « droit » d'exécuter du Go ; il y en a `GOMAXPROCS` |

Chaque P a une file de goroutines prêtes. Un M doit tenir un P pour exécuter une goroutine. Quand
une goroutine **bloque** (lecture réseau, channel, verrou, `time.Sleep`), l'ordonnanceur la met de
côté et en exécute une autre sur le même P — sans appel au noyau. Quand un P n'a plus rien, il
**vole** du travail dans la file d'un autre (*work stealing*). Depuis Go 1.14, une goroutine qui
calcule longtemps sans jamais bloquer est **préemptée** (interrompue de force) pour ne pas
monopoliser son P.

Le développeur ne voit rien de tout cela : il écrit du code **bloquant**, simple à lire
(`n, err := conn.Read(buf)`), et le runtime en fait un programme non bloquant. C'est l'un des
choix de conception les plus importants de Go.

### `GOMAXPROCS`

`GOMAXPROCS` est le nombre de P, donc le nombre de goroutines qui exécutent du code Go **au même
instant**. Par défaut, c'est le nombre de cœurs logiques — et, depuis Go 1.25, la **limite de CPU
du conteneur** si elle est plus basse (un conteneur limité à 2 CPU sur une machine de 64 cœurs
obtient `GOMAXPROCS=2`, ce qui évite des pauses dues au bridage). On le lit avec
`runtime.GOMAXPROCS(0)` et on le change avec la variable d'environnement `GOMAXPROCS`. Le
modifier dans le code est presque toujours une erreur.

### Concurrence ≠ parallélisme

- **Concurrence** : structurer un programme en tâches **indépendantes**, qui *peuvent* progresser
  chacune de leur côté. C'est une question de **conception**.
- **Parallélisme** : exécuter plusieurs choses **au même instant** sur plusieurs cœurs. C'est une
  question d'**exécution**.

Un programme concurrent avec `GOMAXPROCS=1` n'a aucun parallélisme, mais reste concurrent : un
serveur qui attend surtout le réseau gère parfaitement 10 000 connexions sur un seul cœur. À
l'inverse, un calcul pur ne gagne du temps que s'il y a du **parallélisme** — l'exemple de cette
leçon le mesure.

### Piège 1 : `main` n'attend personne

```go
func main() {
	go fmt.Println("bonjour")
}   // le programme se termine ici — la goroutine n'a probablement rien affiché
```

Quand `main` se termine, le programme s'arrête, **toutes** les goroutines avec, sans prévenir et
sans exécuter leurs `defer`. Il faut donc **attendre** explicitement. `time.Sleep` n'est pas une
attente : c'est un pari sur la durée.

L'outil le plus simple pour attendre un groupe de goroutines est `sync.WaitGroup`, détaillé à la
leçon 5. On l'utilise dès maintenant sous sa forme la plus simple (Go 1.25) :

```go
var wg sync.WaitGroup
wg.Go(func() { fmt.Println("bonjour") })   // lance la fonction dans une goroutine et la compte
wg.Wait()                                    // attend que toutes celles lancées par wg.Go soient finies
```

### Piège 2 : ce que la closure capture

```go
for i := range 3 {
	wg.Go(func() { fmt.Println(i) })
}
```

Depuis Go 1.22, chaque tour de boucle a **sa propre** variable `i` (N5 L2) : on obtient bien 0,
1 et 2 — **dans un ordre quelconque**. Mais une closure capture toujours des **variables**, pas
des valeurs : si plusieurs goroutines capturent la **même** variable et que l'une d'elles l'écrit,
c'est une *data race* (leçon 10).

```go
total := 0
for i := range 3 {
	wg.Go(func() { total += i })   // ✗ trois goroutines écrivent la MÊME variable : data race
}
```

Le résultat est imprévisible, et `go run -race` le signale. La solution de l'exemple ci-dessous :
donner à chaque goroutine **sa propre case** pour écrire, et ne lire qu'après le `Wait`.

### L'ordre n'existe pas

Entre deux goroutines, il n'y a **aucun ordre** garanti sans synchronisation explicite. Un
programme qui affiche « toujours » le même ordre sur votre machine peut en afficher un autre en
production, sous charge, ou avec une autre version de Go. Tout raisonnement du type « celle-ci
démarre avant, donc… » est faux.

## Exemple

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

// countPrimes compte les nombres premiers de [lo, hi) — volontairement naïf : du calcul pur.
func countPrimes(lo, hi int) int {
	n := 0
	for x := max(lo, 2); x < hi; x++ {
		prime := true
		for d := 2; d*d <= x; d++ {
			if x%d == 0 {
				prime = false
				break
			}
		}
		if prime {
			n++
		}
	}
	return n
}

// parallel découpe [0, limit) en parts tranches, une goroutine par tranche.
func parallel(limit, parts int) int {
	counts := make([]int, parts) // une case par goroutine : aucune donnée partagée en écriture
	var wg sync.WaitGroup
	step := limit / parts
	for i := range parts {
		lo, hi := i*step, (i+1)*step
		if i == parts-1 {
			hi = limit
		}
		wg.Go(func() { counts[i] = countPrimes(lo, hi) })
	}
	wg.Wait()
	total := 0
	for _, c := range counts {
		total += c
	}
	return total
}

func main() {
	const limit = 3_000_000
	fmt.Println("GOMAXPROCS :", runtime.GOMAXPROCS(0), "— cœurs logiques :", runtime.NumCPU())

	start := time.Now()
	seq := countPrimes(0, limit)
	tSeq := time.Since(start)
	fmt.Printf("séquentiel        : %d premiers en %v\n", seq, tSeq.Round(time.Millisecond))

	for _, parts := range []int{2, 4, 8, 64} {
		start = time.Now()
		got := parallel(limit, parts)
		d := time.Since(start)
		fmt.Printf("%2d goroutines     : %d premiers en %v (×%.1f)\n", parts, got, d.Round(time.Millisecond), float64(tSeq)/float64(d))
	}
}
```

Sortie sur la machine de rédaction (Intel i5-8265U : 4 cœurs physiques, 8 logiques) :

```
GOMAXPROCS : 8 — cœurs logiques : 8
séquentiel        : 216816 premiers en 2.949s
 2 goroutines     : 216816 premiers en 1.686s (×1.7)
 4 goroutines     : 216816 premiers en 944ms (×3.1)
 8 goroutines     : 216816 premiers en 610ms (×4.8)
64 goroutines     : 216816 premiers en 539ms (×5.5)
```

Et avec `GOMAXPROCS=1 go run .` :

```
GOMAXPROCS : 1 — cœurs logiques : 8
séquentiel        : 216816 premiers en 2.74s
 2 goroutines     : 216816 premiers en 2.7s (×1.0)
 4 goroutines     : 216816 premiers en 2.961s (×0.9)
 8 goroutines     : 216816 premiers en 2.755s (×1.0)
64 goroutines     : 216816 premiers en 2.729s (×1.0)
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `counts := make([]int, parts)` | Chaque goroutine écrit **sa** case `counts[i]` : pas de variable partagée en écriture, pas de race. `go run -race .` est silencieux. |
| `lo, hi` calculés **avant** `wg.Go` | Chaque closure capture ses propres `lo`, `hi`, `i` (variables du tour de boucle). |
| `wg.Wait()` avant de lire `counts` | Lire `counts` avant la fin des goroutines serait une race. |
| ×1,7 avec 2, ×3,1 avec 4 | Presque linéaire tant qu'il y a des cœurs **physiques** libres. |
| ×4,8 avec 8 | Seulement 4 cœurs physiques : l'*hyperthreading* n'apporte pas un vrai cœur de plus. |
| ×5,5 avec 64 | Mieux qu'avec 8 : les tranches hautes contiennent des nombres plus grands, donc plus longs à tester. Avec 8 tranches, la dernière finit loin après les autres (**déséquilibre de charge**) ; avec 64 petites tranches, l'ordonnanceur répartit mieux. |
| `GOMAXPROCS=1` : ×1,0 partout | Le programme est toujours **concurrent** (64 goroutines), mais il n'y a plus de **parallélisme** : le calcul ne va pas plus vite. |

## Erreurs fréquentes

1. **Laisser `main` se terminer** sans attendre ses goroutines.
2. **`time.Sleep` pour « attendre »** une goroutine.
3. **Écrire une variable partagée** depuis plusieurs goroutines sans synchronisation.
4. **Supposer un ordre d'exécution** entre goroutines.
5. **Lancer une goroutine sans savoir comment elle se termine** — la fuite de goroutine (leçon 10).
6. **Paralléliser du travail minuscule** : le coût de coordination dépasse le gain.
7. **Croire que plus de goroutines = plus rapide** pour du calcul pur : au-delà du nombre de
   cœurs, on ne gagne qu'en équilibrage de charge.
8. **Modifier `GOMAXPROCS` dans le code** pour « optimiser ».

## Bonnes pratiques Go

- Chaque `go` doit avoir une réponse claire à : **quand et comment cette goroutine se termine-t-elle ?**
- Préférer des fonctions **synchrones** ; laisser l'appelant décider de les lancer en goroutine
  (*Code Review Comments*, « Synchronous Functions »).
- Donner à chaque goroutine ses propres données, ou synchroniser explicitement l'accès partagé.
- Lancer **tout** programme concurrent avec `-race` pendant le développement.
- Mesurer le gain d'une parallélisation avant de la garder.

## Ce que je dois retenir

- `go f()` lance `f` dans une goroutine, sans attendre et sans résultat.
- Une goroutine coûte quelques kilo-octets : on peut en lancer des centaines de milliers.
- Ordonnanceur M:N : goroutines (G) sur threads (M) via `GOMAXPROCS` contextes (P).
- Concurrence = structure ; parallélisme = exécution simultanée.
- `main` qui se termine tue toutes les goroutines.
- Aucun ordre entre goroutines sans synchronisation.
- Une variable écrite par plusieurs goroutines sans protection = *data race*.

---

➡️ [Exercices](exercices.md)
