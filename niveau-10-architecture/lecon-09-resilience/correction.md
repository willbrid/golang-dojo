# Leçon 9 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le code complète le paquet `resil` du cours. `retry.go` est donné en entier (ajout de `RetryAfter`) ;
les mesures des exercices faciles viennent de `corr_test.go`.

```go
// Fichier : resil/retry.go
// Package resil : réessais, disjoncteur, idempotence.
package resil

import (
	"context"
	"errors"
	"math/rand/v2"
	"time"
)

// Backoff calcule l'attente avant la tentative n (n = 1 pour le premier réessai) :
// exponentielle bornée, puis « full jitter » : un tirage uniforme dans [0, plafond].
type Backoff struct {
	Base, Max time.Duration
	Rand      *rand.Rand // injecté : déterministe en test
}

func (b Backoff) Delay(n int) time.Duration {
	ceil := b.Base << (n - 1) // Base × 2^(n-1)
	if ceil > b.Max || ceil <= 0 {
		ceil = b.Max
	}
	return time.Duration(b.Rand.Int64N(int64(ceil) + 1))
}

// Permanent marque une erreur qu'il est inutile de réessayer (400, 404, validation…).
type Permanent struct{ Err error }

func (p Permanent) Error() string { return p.Err.Error() }
func (p Permanent) Unwrap() error { return p.Err }

// RetryAfter : le serveur impose l'attente (429 ou 503 avec Retry-After).
type RetryAfter struct {
	Wait time.Duration
	Err  error
}

func (r RetryAfter) Error() string { return r.Err.Error() }
func (r RetryAfter) Unwrap() error { return r.Err }

var ErrBudget = errors.New("attente imposée supérieure au budget restant")

// Retry exécute op au plus attempts fois. Il s'arrête sur succès, sur erreur permanente,
// ou quand ctx est annulé (le budget global de l'appelant).
func Retry(ctx context.Context, attempts int, b Backoff, sleep func(context.Context, time.Duration) error,
	op func(ctx context.Context) error) error {
	var err error
	for i := 1; i <= attempts; i++ {
		if err = op(ctx); err == nil {
			return nil
		}
		var p Permanent
		if errors.As(err, &p) || i == attempts {
			return err
		}
		wait := b.Delay(i)
		var ra RetryAfter
		if errors.As(err, &ra) {
			wait = ra.Wait
			if dl, ok := ctx.Deadline(); ok && time.Until(dl) < wait {
				return errors.Join(ErrBudget, err) // inutile d'attendre pour échouer ensuite
			}
		}
		if serr := sleep(ctx, wait); serr != nil {
			return errors.Join(serr, err) // budget épuisé : on rend aussi la dernière cause
		}
	}
	return err
}

// Sleep : l'attente réelle, interrompue par l'annulation.
func Sleep(ctx context.Context, d time.Duration) error {
	t := time.NewTimer(d)
	defer t.Stop()
	select {
	case <-ctx.Done():
		return ctx.Err()
	case <-t.C:
		return nil
	}
}
```

```go
// Fichier : resil/corr_test.go
package resil_test

import (
	"context"
	"errors"
	"fmt"
	"io"
	"math/rand/v2"
	"net/http"
	"net/http/httptest"
	"strings"
	"sync"
	"sync/atomic"
	"testing"
	"time"

	"exemple/resil"
)

// E1 : 200 clients contre un serveur indisponible 500 ms, compté par tranche de 50 ms.
func TestE1Herd(t *testing.T) {
	jit := resil.Backoff{Base: 100 * time.Millisecond, Max: 2 * time.Second, Rand: rand.New(rand.NewPCG(3, 4))}
	strategies := map[string]func(n int) time.Duration{
		"a-immédiat": func(int) time.Duration { return 0 },
		"b-fixe":     func(int) time.Duration { return 100 * time.Millisecond },
		"c-expo":     func(n int) time.Duration { return 100 * time.Millisecond << (n - 1) },
		"d-jitter":   jit.Delay,
	}
	for _, name := range []string{"a-immédiat", "b-fixe", "c-expo", "d-jitter"} {
		delay := strategies[name]
		var mu sync.Mutex
		buckets := make([]int, 40)
		start := time.Now()
		srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			el := time.Since(start)
			mu.Lock()
			if b := int(el / (50 * time.Millisecond)); b < len(buckets) {
				buckets[b]++
			}
			mu.Unlock()
			if el < 500*time.Millisecond {
				w.WriteHeader(http.StatusServiceUnavailable)
			}
		}))
		var ok atomic.Int64
		var wg sync.WaitGroup
		var dmu sync.Mutex
		for range 200 {
			wg.Go(func() {
				for n := 1; n <= 5; n++ {
					resp, err := srv.Client().Get(srv.URL)
					if err == nil {
						resp.Body.Close()
						if resp.StatusCode == 200 {
							ok.Add(1)
							return
						}
					}
					dmu.Lock()
					d := delay(n) // le générateur partagé n'est pas sûr en concurrence
					dmu.Unlock()
					time.Sleep(d)
				}
			})
		}
		wg.Wait()
		srv.Close()
		var parts []string
		for _, b := range buckets[:16] {
			parts = append(parts, fmt.Sprint(b))
		}
		t.Logf("%-11s succès=%3d  requêtes/50ms : %s", name, ok.Load(), strings.Join(parts, " "))
	}
}

// E2 : le budget global coupe les réessais.
func TestE2Budget(t *testing.T) {
	b := resil.Backoff{Base: 10 * time.Millisecond, Max: 10 * time.Millisecond, Rand: rand.New(rand.NewPCG(1, 1))}
	for _, perTry := range []time.Duration{0, 50 * time.Millisecond} {
		ctx, cancel := context.WithTimeout(context.Background(), 300*time.Millisecond)
		attempts := 0
		err := resil.Retry(ctx, 5, b, resil.Sleep, func(ctx context.Context) error {
			attempts++
			if perTry > 0 {
				var c context.CancelFunc
				ctx, c = context.WithTimeout(ctx, perTry)
				defer c()
			}
			select { // une opération qui met 100 ms à échouer
			case <-time.After(100 * time.Millisecond):
				return errors.New("échec lent")
			case <-ctx.Done():
				return ctx.Err()
			}
		})
		cancel()
		t.Logf("E2 délai par tentative=%v : %d tentatives, err=%v", perTry, attempts, err)
	}
}

// E3 : Retry-After respecté, ou abandon si hors budget.
func TestE3RetryAfter(t *testing.T) {
	b := resil.Backoff{Base: time.Millisecond, Max: time.Millisecond, Rand: rand.New(rand.NewPCG(1, 1))}
	var waited []time.Duration
	sleep := func(_ context.Context, d time.Duration) error { waited = append(waited, d); return nil }
	n := 0
	err := resil.Retry(context.Background(), 3, b, sleep, func(context.Context) error {
		if n++; n == 1 {
			return resil.RetryAfter{Wait: 2 * time.Second, Err: errors.New("429")}
		}
		return nil
	})
	if err != nil || len(waited) != 1 || waited[0] != 2*time.Second {
		t.Fatalf("err=%v attentes=%v", err, waited)
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	err = resil.Retry(ctx, 3, b, sleep, func(context.Context) error {
		return resil.RetryAfter{Wait: 5 * time.Second, Err: errors.New("503")}
	})
	if !errors.Is(err, resil.ErrBudget) {
		t.Fatalf("err = %v", err)
	}
}

// E4 : la réponse se perd après la création ; le client réessaie.
func TestE4LostResponse(t *testing.T) {
	for _, withKey := range []bool{false, true} {
		var created atomic.Int64
		var first atomic.Bool
		first.Store(true)
		inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			created.Add(1)
			w.WriteHeader(http.StatusCreated)
		})
		var h http.Handler = inner
		if withKey {
			h = resil.Idempotent(inner)
		}
		srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if first.CompareAndSwap(true, false) {
				h.ServeHTTP(httptest.NewRecorder(), r) // l'opération a lieu…
				panic(http.ErrAbortHandler)            // … mais la connexion est coupée sans réponse
			}
			h.ServeHTTP(w, r)
		}))
		var status int
		for range 3 {
			req, _ := http.NewRequest("POST", srv.URL+"/loans", nil)
			req.Header.Set("Idempotency-Key", "k1")
			resp, err := srv.Client().Do(req)
			if err == nil {
				io.Copy(io.Discard, resp.Body)
				resp.Body.Close()
				status = resp.StatusCode
				break
			}
		}
		srv.Close()
		t.Logf("E4 clé honorée=%v : emprunts créés=%d, statut final=%d", withKey, created.Load(), status)
	}
}

// E5 : le disjoncteur dans le temps.
func TestE5BreakerTimeline(t *testing.T) {
	t0 := time.Date(2026, 3, 1, 0, 0, 0, 0, time.UTC)
	now := t0
	br := &resil.Breaker{Threshold: 3, Cooldown: 30 * time.Second, Now: func() time.Time { return now }}
	fail := func() error { return errors.New("panne") }
	ok := func() error { return nil }
	at := func(sec int) { now = t0.Add(time.Duration(sec) * time.Second) }
	step := func(name string, op func() error, wantRefused bool, want resil.State) {
		t.Helper()
		err := br.Do(op)
		if errors.Is(err, resil.ErrOpen) != wantRefused || br.State() != want {
			t.Fatalf("%s : err=%v état=%v", name, err, br.State())
		}
	}
	step("échec 1", fail, false, resil.Closed)
	step("échec 2", fail, false, resil.Closed)
	step("échec 3", fail, false, resil.Open)
	at(29)
	step("29 s", ok, true, resil.Open)
	at(30)
	step("30 s : essai échoué", fail, false, resil.Open)
	at(59)
	step("59 s", ok, true, resil.Open)
	at(61)
	step("61 s : essai réussi", ok, false, resil.Closed)
	step("échec isolé", fail, false, resil.Closed)
}
```

## E1 — Le troupeau, en vrai

Mesuré, 200 clients, serveur en 503 pendant 500 ms, 5 tentatives (requêtes reçues par tranche de 50 ms) :

```
a-immédiat  succès=  0  requêtes/50ms : 110 418 472 0 0 0 0 0 0 0 0 0 0 0 0 0
b-fixe      succès=143  requêtes/50ms : 200 0 95 105 24 72 126 35 143 57 123 20 0 0 0 0
c-expo      succès=200  requêtes/50ms : 145 55 97 103 0 0 59 99 42 0 0 0 0 0 39 100
d-jitter    succès=169  requêtes/50ms : 188 157 105 79 62 61 36 25 28 39 28 24 16 19 8 9
```

- **Immédiat** : 1 000 requêtes en 150 ms sur un serveur qui ne peut pas répondre, et **aucun** succès :
  les clients ont épuisé leurs tentatives avant la fin de la panne. C'est le pire des deux mondes.
- **Fixe** et **exponentiel sans aléa** : des **rafales** synchronisées (100, 103, 99, 100…) séparées de
  silences. Le serveur reçoit toute la charge au même instant, puis rien.
- ***Full jitter*** : la charge **décroît régulièrement** ; le serveur respire. Mais seuls 169 clients sur
  200 ont réussi : avec un tirage dans `[0, plafond]`, l'attente moyenne est la moitié du plafond, et
  certains clients ont fait leurs 5 tentatives avant la fin des 500 ms. Le *jitter* ne dispense pas de
  dimensionner l'ensemble : base, plafond et nombre de tentatives doivent couvrir la durée des pannes
  qu'on veut absorber (ici, une base de 200 ms, ou 6 tentatives). C'est un compromis à mesurer, pas une
  formule magique.

## E2 — Le budget

```
délai par tentative=0s   : 3 tentatives, err=context deadline exceeded
délai par tentative=50ms : 5 tentatives, err=context deadline exceeded
```

Sans délai par tentative, chaque essai dure 100 ms (plus 10 ms d'attente) : la troisième tentative est
interrompue par le budget de 300 ms. Avec un délai de 50 ms par tentative, chaque essai échoue plus vite
et les 5 tentatives tiennent dans le budget. Le délai par tentative transforme une dépendance **lente** en
dépendance **qui échoue vite**, ce qui laisse le temps de réessayer — utile quand la lenteur est passagère
(un nœud surchargé derrière un répartiteur), inutile si toutes les instances sont lentes.

## E3 — `Retry-After`

Voir `Retry` et `TestE3RetryAfter` : l'attente imposée remplace l'exponentielle ; si elle dépasse le budget
restant (`ctx.Deadline()`), `Retry` abandonne **tout de suite** avec `ErrBudget`, au lieu d'attendre 5 s
pour constater ensuite l'échec.

## E4 — Un `POST` réessayé

```
clé honorée=false : emprunts créés=2, statut final=201
clé honorée=true  : emprunts créés=1, statut final=201
```

La coupure est simulée par `panic(http.ErrAbortHandler)` **après** l'exécution : le serveur ferme la
connexion sans réponse, le client voit une erreur réseau et réessaie. Sans clé honorée par le serveur,
deux emprunts. Avec le middleware `Idempotent`, un seul : la seconde requête reçoit la réponse
enregistrée.

## E5 — Le disjoncteur dans le temps

`TestE5BreakerTimeline` vérifie chaque étape : ouvert au 3ᵉ échec ; refus à 29 s ; essai à 30 s, échoué,
réouverture ; refus à 59 s (le refroidissement repart de 30 s) ; essai réussi à 61 s, fermeture ; un échec
isolé ensuite ne rouvre pas (compteur remis à zéro par le succès).

## Exercice intermédiaire — Un client résilient complet

```go
// Fichier : resil/client.go
// Exercice intermédiaire : un client résilient vers le catalogue.
package resil

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"net"
	"net/http"
	"net/url"
	"strconv"
	"sync/atomic"
	"time"
)

var ErrBusy = errors.New("trop d'appels simultanés vers le catalogue")

type Stats struct {
	Calls, Retries, BreakerRefused, BulkheadRefused atomic.Int64
}

type CatalogClient struct {
	Base     string
	HTTP     *http.Client
	PerTry   time.Duration
	Attempts int
	Backoff  Backoff
	Sleep    func(context.Context, time.Duration) error
	Breaker  *Breaker
	slots    chan struct{}
	Stats    Stats
}

func NewCatalogClient(base string, hc *http.Client, b *Breaker, backoff Backoff,
	sleep func(context.Context, time.Duration) error) *CatalogClient {
	return &CatalogClient{Base: base, HTTP: hc, PerTry: 200 * time.Millisecond, Attempts: 3,
		Backoff: backoff, Sleep: sleep, Breaker: b, slots: make(chan struct{}, 10)}
}

func (c *CatalogClient) Title(ctx context.Context, isbn string) (string, error) {
	c.Stats.Calls.Add(1)
	select { // cloison : 10 appels simultanés au plus, refus immédiat au-delà
	case c.slots <- struct{}{}:
		defer func() { <-c.slots }()
	default:
		c.Stats.BulkheadRefused.Add(1)
		return "", ErrBusy
	}
	var title string
	// Le disjoncteur ENTOURE les réessais : il compte un échec par opération (après ses
	// réessais), et un circuit ouvert évite même la première tentative.
	err := c.Breaker.Do(func() error {
		tries := 0
		return Retry(ctx, c.Attempts, c.Backoff, c.Sleep, func(ctx context.Context) error {
			if tries++; tries > 1 {
				c.Stats.Retries.Add(1)
			}
			var err error
			title, err = c.once(ctx, isbn)
			return err
		})
	})
	if errors.Is(err, ErrOpen) {
		c.Stats.BreakerRefused.Add(1)
	}
	return title, err
}

func (c *CatalogClient) once(ctx context.Context, isbn string) (string, error) {
	ctx, cancel := context.WithTimeout(ctx, c.PerTry)
	defer cancel()
	req, err := http.NewRequestWithContext(ctx, "GET", c.Base+"/works/"+url.PathEscape(isbn), nil)
	if err != nil {
		return "", Permanent{err}
	}
	resp, err := c.HTTP.Do(req)
	if err != nil {
		var ne net.Error
		if errors.As(err, &ne) || errors.Is(err, context.DeadlineExceeded) {
			return "", err // passagère
		}
		return "", Permanent{err}
	}
	defer resp.Body.Close()
	switch s := resp.StatusCode; {
	case s == http.StatusOK:
		var body struct{ Title string }
		if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
			return "", Permanent{err}
		}
		return body.Title, nil
	case s == http.StatusTooManyRequests || s == http.StatusServiceUnavailable:
		err := fmt.Errorf("catalogue : %d", s)
		if secs, perr := strconv.Atoi(resp.Header.Get("Retry-After")); perr == nil {
			return "", RetryAfter{Wait: time.Duration(secs) * time.Second, Err: err}
		}
		return "", err
	case s == http.StatusBadGateway || s == http.StatusGatewayTimeout:
		return "", fmt.Errorf("catalogue : %d", s)
	default: // 4xx et autres : inutile de réessayer
		return "", Permanent{fmt.Errorf("catalogue : %d", s)}
	}
}
```

```go
// Fichier : resil/client_test.go
package resil_test

import (
	"context"
	"errors"
	"math/rand/v2"
	"net/http"
	"net/http/httptest"
	"sync"
	"sync/atomic"
	"testing"
	"time"

	"exemple/resil"
)

func newClient(url string, hc *http.Client, now *time.Time) *resil.CatalogClient {
	br := &resil.Breaker{Threshold: 5, Cooldown: 10 * time.Second, Now: func() time.Time { return *now }}
	noWait := func(context.Context, time.Duration) error { return nil }
	return resil.NewCatalogClient(url, hc, br,
		resil.Backoff{Base: time.Millisecond, Max: time.Millisecond, Rand: rand.New(rand.NewPCG(1, 1))}, noWait)
}

func TestClientCases(t *testing.T) {
	var mode atomic.Value
	var hits atomic.Int64
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		hits.Add(1)
		switch mode.Load().(string) {
		case "ok":
			w.Write([]byte(`{"Title":"La Peste"}`))
		case "503":
			w.WriteHeader(503)
		case "404":
			w.WriteHeader(404)
		case "lent":
			time.Sleep(300 * time.Millisecond)
		}
	}))
	defer srv.Close()
	now := time.Now()
	c := newClient(srv.URL, srv.Client(), &now)
	ctx := t.Context()

	mode.Store("ok")
	if got, err := c.Title(ctx, "x"); err != nil || got != "La Peste" {
		t.Fatalf("ok : %q %v", got, err)
	}
	mode.Store("404")
	hits.Store(0)
	if _, err := c.Title(ctx, "x"); err == nil || hits.Load() != 1 {
		t.Fatalf("404 réessayé : %d appels, %v", hits.Load(), err)
	}
	mode.Store("lent")
	hits.Store(0)
	start := time.Now()
	if _, err := c.Title(ctx, "x"); !errors.Is(err, context.DeadlineExceeded) || hits.Load() != 3 {
		t.Fatalf("lent : %d appels, %v", hits.Load(), err)
	}
	if d := time.Since(start); d > time.Second {
		t.Fatalf("délai par tentative non respecté : %v", d)
	}
	// 503 répétés : chaque opération (3 tentatives) compte UN échec pour le disjoncteur.
	mode.Store("503")
	for range 5 {
		c.Title(ctx, "x")
	}
	hits.Store(0)
	if _, err := c.Title(ctx, "x"); !errors.Is(err, resil.ErrOpen) || hits.Load() != 0 {
		t.Fatalf("disjoncteur : %d appels, %v", hits.Load(), err)
	}
	t.Logf("appels=%d réessais=%d refus-disjoncteur=%d", c.Stats.Calls.Load(), c.Stats.Retries.Load(), c.Stats.BreakerRefused.Load())
}

func TestBulkhead(t *testing.T) {
	release := make(chan struct{})
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		<-release
		w.Write([]byte(`{"Title":"t"}`))
	}))
	defer srv.Close()
	now := time.Now()
	c := newClient(srv.URL, srv.Client(), &now)
	c.PerTry = 5 * time.Second
	var wg sync.WaitGroup
	for range 10 {
		wg.Go(func() { c.Title(t.Context(), "x") })
	}
	time.Sleep(100 * time.Millisecond) // les 10 sont en vol
	if _, err := c.Title(t.Context(), "x"); !errors.Is(err, resil.ErrBusy) {
		t.Fatalf("11e appel : %v", err)
	}
	close(release)
	wg.Wait()
}
```

`go test -race ./...` passe, en moins d'une seconde pour ces tests (attentes injectées). Points de
conception :

- **Le disjoncteur autour des réessais** : il compte **une** opération échouée après ses trois tentatives,
  pas trois échecs ; et quand il est ouvert, aucune tentative n'a lieu (`hits=0` dans le test). À
  l'intérieur, il s'ouvrirait trois fois plus vite, et chaque appelant ferait quand même ses réessais
  contre un circuit ouvert.
- **La cloison** refuse immédiatement le onzième appel simultané (`ErrBusy`). Choix discutable :
  une attente bornée (sélection sur `ctx.Done()` et le sémaphore) lisserait les pointes ; le refus
  immédiat protège mieux l'appelant quand la dépendance est lente. Dans les deux cas, c'est le
  sémaphore de N6 L11.
- **Classement des erreurs** : réseau et délai → passagères ; 502/503/504/429 → passagères ; tout autre
  statut → `Permanent`. Un 404 n'est appelé qu'**une** fois.
- **Compteurs** : exposés pour la leçon 10 (`appels=9 réessais=8 refus-disjoncteur=3` dans le test).

## Défi — Des clés d'idempotence pour plusieurs instances

```go
// Fichier : pgidem/pgidem.go
// Défi : des clés d'idempotence partagées entre instances, dans PostgreSQL.
package pgidem

import (
	"bytes"
	"context"
	"database/sql"
	"errors"
	"net/http"
	"net/http/httptest"
	"time"
)

const Schema = `
	CREATE TABLE idempotency (
		client     text NOT NULL,
		key        text NOT NULL,
		status     int,
		body       bytea,
		completed  boolean NOT NULL DEFAULT false,
		created_at timestamptz NOT NULL DEFAULT now(),
		PRIMARY KEY (client, key));
	CREATE TABLE loans (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, client text);`

// Stale : au-delà, une clé « en cours » est considérée abandonnée (l'instance est tombée).
const Stale = 30 * time.Second

var ErrInProgress = errors.New("requête identique toujours en cours")

func Middleware(db *sql.DB, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		key, client := r.Header.Get("Idempotency-Key"), r.Header.Get("X-Client")
		if key == "" || client == "" || r.Method != http.MethodPost {
			next.ServeHTTP(w, r)
			return
		}
		ctx := r.Context()
		// 1. Réserver la clé. L'insertion réussit pour UNE seule requête, toutes instances confondues.
		res, err := db.ExecContext(ctx, `
			INSERT INTO idempotency (client, key) VALUES ($1, $2)
			ON CONFLICT (client, key) DO UPDATE SET created_at = now()
			WHERE NOT idempotency.completed AND idempotency.created_at < now() - make_interval(secs => $3)`,
			client, key, Stale.Seconds())
		if err != nil {
			http.Error(w, "erreur interne", 500)
			return
		}
		if n, _ := res.RowsAffected(); n == 1 { // réservée (ou reprise après abandon) : on exécute
			rec := httptest.NewRecorder()
			next.ServeHTTP(rec, r)
			if _, err := db.ExecContext(context.WithoutCancel(ctx), `
				UPDATE idempotency SET status = $3, body = $4, completed = true
				WHERE client = $1 AND key = $2`, client, key, rec.Code, rec.Body.Bytes()); err != nil {
				http.Error(w, "erreur interne", 500)
				return
			}
			w.WriteHeader(rec.Code)
			w.Write(rec.Body.Bytes())
			return
		}
		// 2. Déjà réservée : attendre la réponse de l'autre exécution (interrogation courte, bornée).
		for {
			var status sql.Null[int]
			var body []byte
			var done bool
			if err := db.QueryRowContext(ctx, `SELECT status, body, completed FROM idempotency
				WHERE client = $1 AND key = $2`, client, key).Scan(&status, &body, &done); err != nil {
				http.Error(w, "erreur interne", 500)
				return
			}
			if done {
				w.Header().Set("Idempotent-Replayed", "true")
				w.WriteHeader(status.V)
				bytes.NewReader(body).WriteTo(w)
				return
			}
			select {
			case <-ctx.Done():
				return
			case <-time.After(20 * time.Millisecond):
			}
		}
	})
}
```

```go
// Fichier : pgidem/pgidem_test.go
package pgidem

import (
	"context"
	"crypto/rand"
	"database/sql"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"os"
	"strings"
	"sync"
	"testing"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func newDB(t *testing.T) *sql.DB {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	admin, _ := sql.Open("pgx", dsn)
	t.Cleanup(func() { admin.Close() })
	schema := "test_" + strings.ToLower(rand.Text()[:12])
	if _, err := admin.ExecContext(t.Context(), "CREATE SCHEMA "+schema); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { admin.ExecContext(context.Background(), "DROP SCHEMA "+schema+" CASCADE") })
	db, _ := sql.Open("pgx", dsn+"&search_path="+schema)
	db.SetMaxOpenConns(20)
	t.Cleanup(func() { db.Close() })
	if _, err := db.ExecContext(t.Context(), Schema); err != nil {
		t.Fatal(err)
	}
	return db
}

func TestTwoInstancesOneExecution(t *testing.T) {
	db := newDB(t)
	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		var id int64
		db.QueryRowContext(r.Context(), `INSERT INTO loans (client) VALUES ($1) RETURNING id`,
			r.Header.Get("X-Client")).Scan(&id)
		time.Sleep(100 * time.Millisecond) // exécution longue : les rejeux arrivent pendant
		w.WriteHeader(http.StatusCreated)
		fmt.Fprintf(w, "emprunt %d", id)
	})
	a := httptest.NewServer(Middleware(db, handler)) // instance A
	b := httptest.NewServer(Middleware(db, handler)) // instance B, même base
	defer a.Close()
	defer b.Close()

	post := func(base, client, key string) string {
		req, _ := http.NewRequest("POST", base+"/loans", nil)
		req.Header.Set("Idempotency-Key", key)
		req.Header.Set("X-Client", client)
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return err.Error()
		}
		defer resp.Body.Close()
		body, _ := io.ReadAll(resp.Body)
		return fmt.Sprintf("%d %s", resp.StatusCode, body)
	}
	var wg sync.WaitGroup
	got := make([]string, 6)
	for i := range 6 {
		wg.Go(func() { got[i] = post([]string{a.URL, b.URL}[i%2], "app-1", "cle-1") })
	}
	wg.Wait()
	other := post(a.URL, "app-2", "cle-1") // même clé, AUTRE client : pas un rejeu

	var n int
	db.QueryRow(`SELECT count(*) FROM loans`).Scan(&n)
	t.Logf("réponses : %q ; autre client : %q ; emprunts en base : %d", got, other, n)
	for _, g := range got {
		if g != got[0] || !strings.HasPrefix(g, "201 emprunt") {
			t.Fatalf("réponses différentes : %q", got)
		}
	}
	if n != 2 || other == got[0] {
		t.Fatalf("emprunts=%d (attendu 2 : un par client)", n)
	}
}

func TestAbandonedKeyIsTakenOver(t *testing.T) {
	db := newDB(t)
	// Une clé réservée il y a 40 s par une instance tombée, jamais terminée.
	db.Exec(`INSERT INTO idempotency (client, key, created_at) VALUES ('app-1', 'k', now() - interval '40 seconds')`)
	srv := httptest.NewServer(Middleware(db, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusCreated)
	})))
	defer srv.Close()
	req, _ := http.NewRequest("POST", srv.URL, nil)
	req.Header.Set("Idempotency-Key", "k")
	req.Header.Set("X-Client", "app-1")
	ctx, cancel := context.WithTimeout(t.Context(), 2*time.Second)
	defer cancel()
	resp, err := http.DefaultClient.Do(req.WithContext(ctx))
	if err != nil || resp.StatusCode != 201 {
		t.Fatalf("clé abandonnée non reprise : %v", err)
	}
	resp.Body.Close()
}
```

Mesuré : six requêtes avec la même clé, réparties sur deux instances (deux serveurs sur la même base),
**une** exécution et six réponses identiques (`201 emprunt 1`) ; la même clé envoyée par un autre client
est une opération distincte (`201 emprunt 2`) ; une clé réservée depuis 40 s et jamais terminée est reprise.

1. **La réservation** est un `INSERT … ON CONFLICT` : la contrainte de clé primaire garantit qu'une seule
   requête, toutes instances confondues, obtient `RowsAffected() == 1`. Les autres attendent `completed`.
2. **L'instance tombée** : une clé non terminée depuis plus de `Stale` (30 s) peut être reprise (la clause
   `WHERE` de l'`ON CONFLICT DO UPDATE`). Risque : si l'opération était **en réalité** encore en cours
   (très lente, pas tombée), elle sera exécutée deux fois. On choisit `Stale` nettement plus long que le
   délai maximal d'une requête, et on accepte un risque résiduel — ou on exécute l'opération et
   l'enregistrement dans la même transaction (point 5).
3. **Lien au client** : la clé primaire est `(client, key)`.
4. **Expiration** : purge par lots des clés de plus de 24 h (même technique que la purge de l'*outbox*,
   leçon 8).
5. **Opération et réponse dans la même transaction** : si l'opération est une écriture en base, on peut
   insérer l'emprunt **et** marquer la clé terminée dans **une** transaction. Alors il est impossible
   d'avoir l'un sans l'autre : une panne entre les deux n'existe plus, et la reprise d'une clé abandonnée
   ne peut plus produire de double exécution (la transaction abandonnée a été annulée avec son emprunt).
   C'est le même raisonnement que l'*outbox* : une seule écriture atomique plutôt que deux.

## Quiz

1. Sans délai, une dépendance bloquée immobilise indéfiniment goroutines et connexions ; aucun autre
   mécanisme ne peut agir.
2. Une erreur passagère, et une opération idempotente.
3. Les erreurs permanentes : 400, 401, 403, 404, 422, validation.
4. Exponentielle : laisser la dépendance récupérer ; aléa : désynchroniser les clients qui ont échoué
   ensemble.
5. Un tirage uniforme de l'attente entre 0 et le plafond exponentiel.
6. Fermé (appels permis, échecs comptés) → ouvert au seuil (refus immédiat) → semi-ouvert après le délai
   (un essai) → fermé si succès, ouvert si échec.
7. Une réponse immédiate (et un mode dégradé) au lieu d'attendre un délai pour échouer.
8. Le serveur mémorise la réponse associée à la clé et la rend sans réexécuter.
9. Attendre la fin de la première exécution, puis rendre sa réponse.
10. Les réessais se multiplient d'un niveau à l'autre et transforment une panne courte en surcharge.
