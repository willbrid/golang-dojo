# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — MinMax

```go
func MinMax(nums ...int) (int, int, error) {
	if len(nums) == 0 {
		return 0, 0, errors.New("aucune valeur")
	}
	lo, hi := nums[0], nums[0]
	for _, n := range nums[1:] {
		lo = min(lo, n) // min et max sont intégrés depuis Go 1.21
		hi = max(hi, n)
	}
	return lo, hi, nil
}
```

**Pourquoi une erreur plutôt que `0, 0` ?** Parce que `0, 0` est indiscernable d'un vrai
résultat sur `MinMax(0)`. Retourner une erreur rend l'absence de données **impossible à
ignorer par accident** — le compilateur force à déclarer la variable, le linter signale si
elle n'est pas testée.

## E2 — Variadique

```go
func Sum(nums ...int) int { … }

Sum()             // 0     : nums est un slice nil, range fait zéro tour
Sum(1, 2, 3)      // 6
Sum(xs...)        // étalement
```

```go
func Corrupt(nums ...int) {
	if len(nums) > 0 {
		nums[0] = 999
	}
}
xs := []int{1, 2, 3}
Corrupt(xs...)
fmt.Println(xs)   // [999 2 3]  ← l'original est modifié !
```

`f(xs...)` passe un slice qui **partage le tableau sous-jacent** de `xs`. Aucune copie.
En revanche `Corrupt(1, 2, 3)` construit un slice temporaire : rien à corrompre.

C'est un piège réel : une fonction variadique qui modifie son paramètre doit le documenter.

## E3 — Compteur

```go
func Counter() func() int {
	n := 0
	return func() int {
		n++
		return n
	}
}

c1, c2 := Counter(), Counter()
fmt.Println(c1(), c1(), c1(), c2())  // 1 2 3 1
```

`n` est déclaré dans `Counter` mais référencé par la closure retournée. Le compilateur
détecte qu'il **s'échappe** de la fonction et l'alloue sur le **tas**, pas sur la pile. Le
ramasse-miettes le libérera quand plus aucune closure ne le référencera. Chaque appel à
`Counter()` crée un `n` distinct.

## E4 — Apply

```go
func Apply(nums []int, f func(int) int) []int {
	out := make([]int, len(nums)) // NOUVEAU slice : l'original est préservé
	for i, n := range nums {
		out[i] = f(n)
	}
	return out
}

doubled := Apply(xs, func(n int) int { return n * 2 })
```

L'erreur à éviter : `for i := range nums { nums[i] = f(nums[i]) }` modifie l'entrée. La
contrainte « l'original ne doit pas être modifié » impose l'allocation.

## E5 — Masquage

Le bug est à la ligne `cfg, err := applyDebug(cfg)` : le `:=` déclare **deux nouvelles
variables** `cfg` et `err` locales au bloc `if`. Le `cfg` transformé n'existe que dans ce
bloc ; à la sortie, `use(cfg)` reçoit la **configuration d'origine**, non transformée.

Le compilateur ne dit rien : les deux variables sont bien utilisées à l'intérieur du bloc.

Correction :

```go
if cfg.Debug {
	var err error
	cfg, err = applyDebug(cfg)   // = et non := : on réaffecte les variables externes
	if err != nil {
		return err
	}
	fmt.Println(cfg.Level)
}
```

Détecté par `golangci-lint` avec le linter `shadow`, pas par `go vet` seul.

## Exercice intermédiaire — `Retry`

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

// Retry exécute op jusqu'à attempts fois, en attendant delay entre deux essais.
// Elle ne journalise rien : une bibliothèque ne doit pas décider du format des logs
// de son appelant. Pour observer, l'appelant enveloppe op dans sa propre closure.
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
		if i < attempts-1 { // pas d'attente APRÈS le dernier essai
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
	calls := 0
	flaky := func() error { // closure sur calls : c'est le point de l'exercice
		calls++
		if calls < 3 {
			return fmt.Errorf("tentative %d : service indisponible", calls)
		}
		return nil
	}

	if err := Retry(5, 100*time.Millisecond, flaky); err != nil {
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

**Sur la contrainte 6 (journalisation).** Une fonction de bibliothèque ne doit pas imposer
son format de logs. Deux options propres : soit l'appelant enveloppe `op` lui-même, soit on
accepte un `onRetry func(attempt int, err error)` optionnel. Décider et documenter valait
autant que le code.

**Ce qui manque encore** : ce `Retry` est **non annulable**. Si l'utilisateur ferme sa
connexion, on continue à dormir. La version production prend un `context.Context` en premier
paramètre et utilise `select` sur `ctx.Done()` — niveau 4. Y ajouter un *jitter* aléatoire
évite que mille clients réessaient à la même milliseconde — niveau 7.

## Défi — Compose

```go
type Transform func(string) string

// Compose retourne une Transform appliquant ts dans l'ordre.
// Le slice est copié à la construction : modifier ts après coup n'affecte
// pas la transformation déjà construite (voir la question (d)).
func Compose(ts ...Transform) Transform {
	fns := slices.Clone(ts)
	return func(s string) string {
		for _, t := range fns {
			s = t(s)
		}
		return s
	}
}
```

`Compose()` sans argument retourne une closure sur un slice vide : la boucle ne s'exécute
pas et `s` ressort inchangé. C'est **l'identité**, et non `nil` : la contrainte (b) est
satisfaite sans cas particulier. Une conception qui rend le cas dégénéré correct « par
construction » est toujours préférable à un `if len(ts) == 0`.

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

**(d)** Sans `slices.Clone`, la closure capture le slice `ts` — un descripteur qui pointe
vers le tableau de l'appelant. Si celui-ci fait `ts[0] = autreChose`, la transformation
déjà « construite » change de comportement à distance. C'est un bug d'aliasing (leçon 6)
déguisé en question de conception. Copier au moment de la construction rend la valeur
retournée **immuable**, ce qui est presque toujours le bon contrat pour une valeur qu'on
distribue.

## Réponses du quiz

1. En **dernier**. Convention universelle : elle rend le motif `v, err := f()` reconnaissable
   d'un coup d'œil, et les outils s'appuient dessus.
2. Un slice `nil` de longueur 0. `range` et `len` fonctionnent normalement.
3. **Non.** Le slice passé partage le tableau sous-jacent de `xs`.
4. **La variable elle-même**, par référence. C'est pourquoi elle peut survivre à la fonction
   qui l'a déclarée.
5. Sur le **tas**. Le compilateur le détermine par *escape analysis*.
6. Dans une fonction courte, où les résultats nommés documentent la signature. Au-delà de
   quelques lignes, il nuit à la lisibilité.
7. Panique : `nil pointer dereference` (une valeur de fonction nil n'a pas de code à appeler).
8. Parce que `:=` déclare de **nouvelles** variables dans le bloc courant. L'erreur affectée
   à la variable interne est perdue à la sortie du bloc, et le compilateur ne proteste pas.
9. Autant qu'on veut — mais au-delà de trois, une struct nommée est presque toujours plus
   lisible.
