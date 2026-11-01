# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le hasard de `select`

Chaque cas est choisi environ **500** fois sur 1 000 (par exemple 487 / 513, 509 / 491…), sans
préférence pour le premier écrit. La spécification l'impose : *« a single one is chosen via a
uniform pseudo-random selection »*. Ne jamais s'appuyer sur l'ordre des `case` pour donner une
priorité ; pour prioriser, il faut deux `select` imbriqués (tester d'abord le canal prioritaire avec
un `default`, puis attendre tous les canaux).

## E2 — Délai

```go
var ErrTimeout = errors.New("délai dépassé")

func FirstOf(timeout time.Duration, srcs ...func() string) (string, error) {
	res := make(chan string, len(srcs))   // capacité = nombre d'émetteurs : aucun ne bloquera
	for _, f := range srcs {
		go func() { res <- f() }()
	}
	select {
	case v := <-res:
		return v, nil
	case <-time.After(timeout):
		return "", ErrTimeout
	}
}
```

La capacité `len(srcs)` est la clé : les perdants déposent leur résultat dans le buffer et se
terminent. Avec un channel non bufferisé, `len(srcs) - 1` goroutines resteraient bloquées pour
toujours — `runtime.NumGoroutine()` le montre après une pause.

## E3 — Le `default` qui brûle le CPU

`time go run .` affiche typiquement `real 2,0 s` et `user 2,0 s` : un cœur entier occupé pendant
toute l'attente, à faire tourner la boucle à vide. Correction : retirer le `default`. Le `select`
bloque alors, la goroutine est mise en sommeil par l'ordonnanceur, et `user` tombe à quelques
millisecondes.

## E4 — Le channel fermé toujours prêt

Sans `a = nil`, dès que `a` est fermé, son cas est **toujours prêt** (réception immédiate de la
zéro-valeur avec `ok == false`). La boucle tourne alors des milliers, voire des millions de fois,
jusqu'à ce que `b` soit aussi fermé… et si la condition de boucle porte sur `a != nil || b != nil`,
elle ne se termine **jamais** : `a` n'est jamais remis à nil. Attente active infinie à 100 % de CPU.

## E5 — Arrêter plusieurs goroutines

Un **envoi** sur `quit` n'est reçu que par **une** goroutine : une seule s'arrête, les quatre
autres continuent (et `wg.Wait()` bloquerait). La **fermeture** est reçue par toutes : les cinq
s'arrêtent, `NumGoroutine()` revient à 1. C'est pourquoi la diffusion d'un signal d'arrêt passe
toujours par `close` — et c'est exactement ce que fait `context` (leçon 8).

## Exercice intermédiaire — Un limiteur de débit

```go
type Limiter struct {
	tokens chan struct{}
	quit   chan struct{}
	done   chan struct{}
}

func NewLimiter(perSecond int) *Limiter {
	l := &Limiter{
		tokens: make(chan struct{}, perSecond),
		quit:   make(chan struct{}),
		done:   make(chan struct{}),
	}
	for range perSecond {            // seau plein au départ : rafale autorisée
		l.tokens <- struct{}{}
	}
	go l.refill(time.Second / time.Duration(perSecond))
	return l
}

func (l *Limiter) refill(every time.Duration) {
	defer close(l.done)
	t := time.NewTicker(every)
	defer t.Stop()
	for {
		select {
		case <-t.C:
			select {
			case l.tokens <- struct{}{}:  // un jeton de plus…
			default:                       // …sauf si le seau est plein : on ne bloque pas
			}
		case <-l.quit:
			return
		}
	}
}

func (l *Limiter) Wait() { <-l.tokens }

func (l *Limiter) TryAcquire() bool {
	select {
	case <-l.tokens:
		return true
	default:
		return false
	}
}

func (l *Limiter) Stop() {
	close(l.quit)
	<-l.done
}
```

- Le `select` imbriqué avec `default` empêche la goroutine de recharge de bloquer quand personne ne
  consomme : sinon elle ne pourrait plus voir `quit`.
- 100 appels à 20/s : les **20 premiers** sont immédiats (seau plein), puis un jeton toutes les
  50 ms pour les 80 suivants, soit **environ 4 secondes**.
- `Wait` sur un limiteur arrêté bloque pour toujours : à documenter, ou faire rendre une erreur à
  `Wait` (avec un `select` sur `quit`).
- En production, on utilise `golang.org/x/time/rate`, qui calcule les jetons sans goroutine ni
  ticker. Écrire celui-ci sert à comprendre `select`.

Test avec `synctest` :

```go
func TestLimiterRate(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		l := NewLimiter(20)
		defer l.Stop()
		start := time.Now()
		for range 100 {
			l.Wait()
		}
		if got := time.Since(start); got != 4*time.Second {
			t.Errorf("100 jetons en %v, attendu 4s", got)
		}
	})
}
```

En temps virtuel, la mesure est **exacte** : 80 jetons × 50 ms = 4 s pile. Le test dure quelques
millisecondes réelles.

## Défi — Tester le temps sans l'attendre : `testing/synctest`

```go
package s

import (
	"errors"
	"testing"
	"testing/synctest"
	"time"
)

func TestWithTimeout(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		start := time.Now()
		_, err := withTimeout(fetch(time.Hour, "x"), 2*time.Second)
		if !errors.Is(err, ErrTimeout) {
			t.Fatalf("err = %v", err)
		}
		if el := time.Since(start); el != 2*time.Second {
			t.Errorf("écoulé = %v", el)
		}
		v, err := withTimeout(fetch(time.Second, "ok"), 2*time.Second)
		if err != nil || v != "ok" {
			t.Errorf("%q %v", v, err)
		}
		// La goroutine du premier fetch dort encore (1 h virtuelle) : sans cette ligne,
		// synctest échoue en signalant une goroutine bloquée. On laisse le temps virtuel
		// avancer jusqu'à ce qu'elle se termine.
		time.Sleep(time.Hour)
		synctest.Wait()
	})
}
```

1. Le test vérifie `time.Since(start) == 2*time.Second` **exactement** : dans la bulle, le temps
   n'avance que quand toutes les goroutines sont bloquées, et il saute directement à la prochaine
   échéance.
2. Sans les deux dernières lignes, le test échoue (mesuré) :

   ```
   panic: deadlock: main bubble goroutine has exited but blocked goroutines remain
   ```

   La goroutine encore bloquée est celle du **premier** `fetch(time.Hour, …)` : elle dort encore
   58 minutes virtuelles. `synctest` vient de révéler précisément ce que le cours affirmait —
   **abandonner l'attente n'arrête pas le travail**. Dans un vrai serveur, chaque délai dépassé
   laisserait une goroutine travailler pour rien.
3. Ajouter `time.Sleep(time.Hour)` puis `synctest.Wait()` à la fin de la bulle laisse le temps
   virtuel avancer jusqu'à la fin de cette goroutine. C'est légitime **ici** parce que `fetch`
   termine bel et bien (elle ne fuit pas, elle est seulement lente) : on vérifie qu'elle finit.
   Si elle ne terminait jamais, aucun `Sleep` ne la débloquerait, et l'échec serait le bon verdict.
4. Il faudrait que `fetch` **reçoive un signal d'annulation** et l'écoute pendant son travail
   (`select` sur ce signal et sur la fin du travail) : c'est exactement ce que fournit
   `context.Context` (leçon 8).

## Quiz

1. Il en choisit un au hasard, uniformément.
2. Il bloque jusqu'à ce qu'un cas devienne prêt.
3. `select` avec un `case <-time.After(d)`.
4. Non : seule l'attente est abandonnée ; il faut un mécanisme d'annulation (`context`).
5. Il rend une opération non bloquante ; dangereux dans une boucle d'attente (100 % de CPU).
6. Parce qu'un channel fermé est toujours prêt : sans `nil`, son cas serait choisi sans cesse.
7. Une fermeture est reçue par tous les récepteurs ; un envoi par un seul.
8. L'arrêter avec `Stop()`, en général par `defer`.
9. Il bloque pour toujours.
10. Une horloge virtuelle : tester délais, tickers et attentes exactement et instantanément, et
    détecter les goroutines qui restent bloquées.
