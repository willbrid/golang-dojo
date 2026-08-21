# Leçon 11 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Fonction en variable

```go
var op func(int, int) int
op(2, 3)   // panic: runtime error: invalid memory address or nil pointer dereference
```

La zéro-valeur d'un type fonction est `nil`, et une valeur `nil` n'a pas de code à exécuter.
La garde :

```go
if op != nil {
	fmt.Println(op(2, 3))
}
```

À retenir pour tout **rappel optionnel** : un champ `OnError func(error)` doit être testé avant
d'être appelé, ou documenté comme obligatoire.

## E2 — Ordre supérieur

```go
func Apply(nums []int, f func(int) int) []int {
	out := make([]int, len(nums)) // NOUVEAU slice : l'entrée est préservée
	for i, n := range nums {
		out[i] = f(n)
	}
	return out
}

func Filter(nums []int, keep func(int) bool) []int {
	out := make([]int, 0, len(nums)) // len 0, capacité maximale : zéro réallocation
	for _, n := range nums {
		if keep(n) {
			out = append(out, n)
		}
	}
	return out
}
```

Deux préallocations différentes, pour deux raisons différentes : `Apply` connaît la taille
exacte du résultat, `Filter` ne connaît que son maximum.

L'erreur à éviter : `for i := range nums { nums[i] = f(nums[i]) }` modifie l'entrée. La
vérification explicite (`fmt.Println(original)` après l'appel) fait partie de l'exercice —
c'est ainsi qu'on découvre un effet de bord non voulu.

## E3 — Compteurs

```go
func Counter() func() int {
	n := 0
	return func() int { n++; return n }
}

func CounterFrom(start, step int) func() int {
	n := start - step
	return func() int { n += step; return n }
}

c1, c2 := Counter(), Counter()
fmt.Println(c1(), c1(), c1(), c2())   // 1 2 3 1
```

`n` est déclaré dans `Counter` mais référencé après son retour. Le compilateur détecte par
*escape analysis* qu'il **s'échappe** et l'alloue sur le **tas** au lieu de la pile. Le
ramasse-miettes le libérera quand plus aucune closure ne le référencera.

Chaque appel à `Counter()` crée un `n` distinct : deux compteurs sont indépendants. C'est
exactement ce qui manquait au `fibCache` global de la leçon 9.

Vérifiable soi-même :
```bash
go build -gcflags="-m" .    # affiche « moved to heap: n »
```

## E4 — Le piège de la capture

```
A (Go 1.22+) : 012
B            : 222
```

En **A**, la variable de boucle `i` est recréée à chaque itération depuis Go 1.22 : chaque
closure capture la sienne. *(Avant 1.22, A affichait `333`.)*

En **B**, `x` est déclarée **hors** de la boucle : il n'y a qu'une seule variable, capturée par
les trois closures. Après la boucle, elle vaut 2 — les trois affichent donc 2.

**Le correctif de Go 1.22 ne concerne que la variable de boucle.** Le piège de la capture
reste entier pour toute variable partagée, et c'est le cas le plus fréquent dans du vrai code :
un accumulateur, un compteur, un `err` réutilisé.

## E5 — Map de fonctions

```go
var ops = map[string]func(float64, float64) (float64, error){
	"+": func(a, b float64) (float64, error) { return a + b, nil },
	"-": func(a, b float64) (float64, error) { return a - b, nil },
	"*": func(a, b float64) (float64, error) { return a * b, nil },
	"/": func(a, b float64) (float64, error) {
		if b == 0 {
			return 0, errors.New("division par zéro")
		}
		return a / b, nil
	},
}

func compute(op string, a, b float64) (float64, error) {
	f, ok := ops[op]
	if !ok {
		return 0, fmt.Errorf("opérateur inconnu : %q", op)
	}
	return f(a, b)
}
```

| | `switch` | map de fonctions |
|---|---|---|
| Extension | modifier la fonction | ajouter une entrée, éventuellement depuis un autre fichier |
| Lisibilité | excellente pour 4 cas | meilleure au-delà d'une dizaine |
| Coût | comparaisons directes | hachage + appel indirect |
| Vérification | exhaustivité visible | une entrée manquante ne se voit pas |
| Enregistrement dynamique | impossible | naturel (plugins, commandes) |

Pour quatre opérateurs, le `switch` reste préférable. La map devient intéressante quand les
entrées sont **nombreuses**, ou quand elles doivent pouvoir être **enregistrées** par du code
extérieur — c'est ainsi que fonctionnent les registres de commandes CLI et de pilotes SQL.

## Exercice intermédiaire — `retry`

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

// Retry exécute op jusqu'à attempts fois, en attendant delay entre deux essais.
// Elle ne journalise RIEN : une bibliothèque ne doit pas imposer son format de
// logs à l'appelant. Qui veut observer enveloppe op dans sa propre closure —
// c'est précisément ce que les types fonction rendent trivial (voir main).
func Retry(attempts int, delay time.Duration, op func() error) error {
	if attempts <= 0 {
		return fmt.Errorf("attempts doit être positif, reçu %d", attempts)
	}
	var lastErr error
	for i := range attempts {
		if err := op(); err == nil {
			return nil
		} else {
			lastErr = err
		}
		if i < attempts-1 { // AUCUNE attente après le dernier essai
			time.Sleep(delay)
		}
	}
	return fmt.Errorf("échec après %d tentatives : %w", attempts, lastErr)
}

// RetryBackoff double le délai après chaque échec.
func RetryBackoff(attempts int, initial time.Duration, op func() error) error {
	if attempts <= 0 {
		return fmt.Errorf("attempts doit être positif, reçu %d", attempts)
	}
	var lastErr error
	delay := initial
	for i := range attempts {
		if err := op(); err == nil {
			return nil
		} else {
			lastErr = err
		}
		if i < attempts-1 {
			time.Sleep(delay)
			delay *= 2
		}
	}
	return fmt.Errorf("échec après %d tentatives : %w", attempts, lastErr)
}

func main() {
	// L'opération instable : une CLOSURE sur un compteur. C'est le point de
	// l'exercice — sans closure, il faudrait une variable globale.
	calls := 0
	flaky := func() error {
		calls++
		if calls < 3 {
			return fmt.Errorf("tentative %d : service indisponible", calls)
		}
		return nil
	}

	// Observation SANS que Retry ne journalise : on décore op côté appelant.
	traced := func() error {
		err := flaky()
		fmt.Printf("  appel %d → %v\n", calls, err)
		return err
	}

	if err := Retry(5, 100*time.Millisecond, traced); err != nil {
		fmt.Println("échec :", err)
	} else {
		fmt.Printf("succès après %d appels\n", calls)
	}

	always := func() error { return errors.New("panne totale") }
	err := Retry(3, 50*time.Millisecond, always)
	fmt.Println(err)
	fmt.Println("cause préservée :", errors.Unwrap(err))
}
```

**Contrainte 2 — le timing.** `Retry(3, 1s, …)` qui échoue doit durer **2 s**, pas 3. Le test
`i < attempts-1` évite l'attente inutile après la dernière tentative. C'est une erreur
fréquente, invisible en test rapide et coûteuse en production : sur une API appelée un million
de fois par jour avec 1 % d'échec, cela représente des heures d'attente gaspillées.

**Contrainte 6 — la journalisation.** La bonne réponse est **de ne pas journaliser**. Trois
raisons : le format de logs appartient à l'application, pas à la bibliothèque ; ajouter un
paramètre `logger` impose une dépendance à tous les appelants ; et le décorateur côté appelant
(la closure `traced`) résout le problème sans rien coûter. Une alternative acceptable est un
rappel optionnel `onRetry func(attempt int, err error)`, testé contre `nil`.

**Ce qui manque encore.** Ce `Retry` est **non annulable** : si l'utilisateur ferme sa
connexion, on continue de dormir. La version production prend un `context.Context` en premier
paramètre et remplace `time.Sleep` par un `select` sur `ctx.Done()` — niveau 6. Elle ajoute
aussi un **jitter** aléatoire, pour éviter que mille clients ne réessaient à la même
milliseconde et n'achèvent le service qu'ils attendent — niveau 10.

## Défi

**a) `Compose` et `ComposeErr`**

```go
type Transform func(string) string

func Compose(ts ...Transform) Transform {
	fns := slices.Clone(ts) // voir (b)
	return func(s string) string {
		for _, t := range fns {
			s = t(s)
		}
		return s
	}
}
```

`Compose()` sans argument retourne une closure sur un slice vide : la boucle ne s'exécute pas
et `s` ressort inchangé. C'est **l'identité**, obtenue **sans cas particulier**. Une conception
qui rend le cas dégénéré correct par construction vaut toujours mieux qu'un
`if len(ts) == 0 { return identity }`.

```go
func ComposeErr(ts ...func(string) (string, error)) func(string) (string, error) {
	fns := slices.Clone(ts)
	return func(s string) (string, error) {
		var err error
		for i, t := range fns {
			if s, err = t(s); err != nil {
				return "", fmt.Errorf("étape %d : %w", i+1, err)
			}
		}
		return s, nil
	}
}
```

Retourner `""` et non `s` en cas d'erreur : quand l'erreur est non nulle, les autres résultats
ne doivent pas être utilisés (leçon 10).

**b) Aliasing de closure**

Sans `slices.Clone`, la closure capture le **descripteur** du slice `ts`, qui pointe vers le
tableau de l'appelant :

```go
ts := []Transform{strings.ToUpper}
f := Compose(ts...)
fmt.Println(f("abc"))        // ABC
ts[0] = strings.ToLower      // on modifie le slice APRÈS la construction
fmt.Println(f("ABC"))        // abc ← le comportement de f a changé à distance !
```

C'est un bug d'aliasing (leçon 6) déguisé en question de conception. Copier au moment de la
construction rend la valeur retournée **immuable**, ce qui est presque toujours le bon contrat
pour une valeur qu'on distribue à d'autres : personne ne s'attend à ce qu'une fonction change
de comportement sans avoir été reconstruite.

Le cas inverse — la chaîne construite paresseusement, au moment de l'appel — n'a de sens que si
l'on veut délibérément un pipeline **modifiable à chaud**. C'est un besoin réel (rechargement
de configuration), mais il doit alors être documenté et protégé en concurrence (niveau 6).

**c) Options fonctionnelles**

```go
type Server struct {
	host     string
	port     int
	timeout  time.Duration
	maxConns int
	tls      bool
}

type Option func(*Server) error   // ← noter le error

func WithPort(p int) Option {
	return func(s *Server) error {
		if p < 1 || p > 65535 {
			return fmt.Errorf("port hors bornes : %d", p)
		}
		s.port = p
		return nil
	}
}

func NewServer(opts ...Option) (*Server, error) {
	s := &Server{host: "localhost", port: 8080, timeout: 30 * time.Second, maxConns: 100}
	for i, opt := range opts {
		if err := opt(s); err != nil {
			return nil, fmt.Errorf("option %d : %w", i+1, err)
		}
	}
	if s.tls && s.port == 80 { // validation CROISÉE : impossible option par option
		return nil, errors.New("TLS sur le port 80")
	}
	return s, nil
}
```

**Les trois conceptions comparées :**

| | `func(*Server)` | `func(*Server) error` | validation groupée à la fin |
|---|---|---|---|
| Erreur détectée | jamais (ou panique) | au plus tôt, avec le nom de l'option | après application de tout |
| Message | — | « option 2 : port hors bornes » | « configuration invalide » |
| Validations croisées | impossible | impossible | naturelle |
| Verbosité | minimale | moyenne | minimale |

**La bonne réponse combine les deux dernières** : `Option` retourne une `error` pour la
validation **locale** (un port valide est un port valide, indépendamment du reste), et le
constructeur effectue en plus les validations **croisées** qu'aucune option ne peut faire
seule. C'est exactement ce que fait `grpc.NewServer`… à une nuance près : gRPC a choisi
`func(*serverOptions)` sans erreur et valide tout à la fin, ce qui rend ses messages d'erreur
moins précis mais son API plus légère. Les deux choix sont défendables ; savoir dire lequel on
prend et pourquoi est ce qui compte.

## Réponses du quiz

1. `func(int, int) int` — la signature seule, sans les noms de paramètres.
2. Panique : `invalid memory address or nil pointer dereference`.
3. **La variable elle-même**, par référence. C'est pourquoi elle survit à la fonction qui l'a
   déclarée.
4. Sur le **tas**. C'est le **compilateur** qui décide, par *escape analysis* — visible avec
   `go build -gcflags="-m"`.
5. Parce que le correctif ne concerne que la **variable de boucle**. Dans le cas B, `x` est
   déclarée à l'extérieur : il n'y a qu'une seule variable, partagée par toutes les closures.
6. Lisibilité des signatures, documentation (`go doc` affiche le type nommé), et possibilité de
   lui attacher des **méthodes**.
7. En déclarant une méthode sur le type fonction. L'exemple célèbre est
   `http.HandlerFunc`, dont la méthode `ServeHTTP` se contente d'appeler la fonction —
   ce qui permet à une simple fonction de satisfaire `http.Handler`.
8. Une fonction qui en enveloppe une autre en conservant sa signature, pour ajouter un
   comportement transversal. C'est la structure exacte des **middlewares HTTP**
   (`func(http.Handler) http.Handler`) du niveau 7.
9. Parce que Go n'a **ni arguments par défaut, ni arguments nommés**. Les options
   fonctionnelles offrent les deux, tout en restant extensibles sans casser les appels
   existants.
10. **Oui**, une closure alloue : elle porte un pointeur vers son environnement capturé. Ce
    n'est **pas** une raison de l'éviter — sauf dans une boucle exécutée des millions de fois,
    et seulement après l'avoir mesuré (niveau 11).
