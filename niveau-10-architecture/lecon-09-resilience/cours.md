# Leçon 9 — Résilience

> Dès qu'un service en appelle un autre (une base, une API, un courtier), il hérite de ses pannes. Une
> dépendance lente fait s'accumuler les requêtes en attente ; une dépendance tombée fait tomber, en
> cascade, tout ce qui l'appelle sans précaution ; et des réessais mal faits transforment une panne de
> deux secondes en dix minutes de saturation. Cette leçon donne les quatre outils de base — **délais,
> réessais avec *jitter*, disjoncteur, idempotence** — et surtout les règles qui décident quand les
> utiliser.

## Objectifs

1. Donner un **budget de délai** à chaque opération, et le propager.
2. Réessayer **seulement** ce qui peut l'être : opérations idempotentes, erreurs passagères.
3. Calculer les attentes : exponentielle bornée et *full jitter*.
4. Écrire un **disjoncteur** (fermé, ouvert, semi-ouvert).
5. Rendre un `POST` rejouable avec une **clé d'idempotence**.
6. Connaître les cloisons (*bulkheads*) et le mode dégradé.

## Explication

### D'abord, des délais

Sans délai, un appel à une dépendance qui ne répond plus attend **indéfiniment** ; la goroutine, sa
connexion, la requête HTTP entrante qui l'attend restent bloquées, et le service tombe à court de
ressources sans erreur visible (N8 L5 pour le même mécanisme côté base). Règles :

- **chaque appel sortant a un délai**, porté par le `context` (N6 L8, N7 L7) ;
- le délai d'un appel est **plus court** que le délai de la requête qui l'a déclenché : une requête HTTP de
  2 s ne peut pas attendre 3 s une dépendance ;
- avec des réessais, il y a **deux** délais : par tentative (court) et global (le budget de l'appelant).
  `Retry` s'arrête dès que le contexte global est annulé.

### Réessayer, mais pas n'importe quoi

On réessaie une opération quand **deux** conditions sont réunies :

1. **L'erreur est passagère** : délai dépassé, connexion refusée ou coupée, `503 Service Unavailable`,
   `429 Too Many Requests` (en respectant `Retry-After`), `40001`/`40P01` en base (N8 L4). Pas un `400`, un
   `404`, une erreur de validation ou d'autorisation : réessayer donnera le même résultat. Le type
   `Permanent` de l'exemple le dit explicitement.
2. **L'opération est idempotente** : la répéter a le même effet que l'exécuter une fois. Un `GET`, un `PUT`,
   un `DELETE` le sont par définition HTTP. Un `POST` (« créer un emprunt ») ne l'est **pas** : si la
   première tentative a réussi mais que la réponse s'est perdue, la réessayer crée un second emprunt. Il
   faut alors une **clé d'idempotence** (voir plus bas).

Le nombre de tentatives est **borné** (3 à 5) ; au-delà, la panne n'est pas passagère.

### Attendre : exponentielle et *jitter*

Entre deux tentatives, on attend de plus en plus longtemps (exponentielle : 100 ms, 200 ms, 400 ms…, bornée
à quelques secondes), pour laisser la dépendance récupérer. Et on **tire au hasard** l'attente dans
`[0, plafond]` (*full jitter*). Pourquoi l'aléa : quand une dépendance tombe, **tous** ses clients échouent
au même instant ; sans aléa, ils réessaient tous **au même instant** 100 ms plus tard, la refont tomber, et
recommencent ensemble 200 ms plus tard — un troupeau qui piétine (*thundering herd*). Simulé dans
l'exemple, 100 clients :

| | Tranches de 10 ms occupées | Pic de réessais simultanés |
|---|---|---|
| sans *jitter* | 1 | **100** |
| *full jitter* | 10 | 13 |

L'aléa étale la charge des réessais sur toute la fenêtre. L'exemple injecte le générateur aléatoire
(`rand.New(rand.NewPCG(1, 2))`) et la fonction d'attente, pour que le test soit **déterministe** et
instantané.

### Le disjoncteur

Quand une dépendance est réellement en panne, même des réessais bien espacés la sollicitent pour rien, et
chaque appelant attend son délai avant d'échouer. Le **disjoncteur** (*circuit breaker*) coupe le circuit :

```
            échecs consécutifs ≥ seuil
  FERMÉ ─────────────────────────────────► OUVERT ──── refus immédiat (ErrOpen)
    ▲                                        │
    │ essai réussi                           │ délai de refroidissement écoulé
    │                                        ▼
    └───────────────────────────────── SEMI-OUVERT ── un seul appel d'essai
                 essai échoué ─────────► OUVERT
```

- **Fermé** : les appels passent ; on compte les échecs consécutifs.
- **Ouvert** : on refuse **sans essayer**, instantanément. L'appelant passe tout de suite en mode dégradé
  (valeur en cache, réponse partielle, message « service momentanément indisponible ») au lieu d'attendre.
- **Semi-ouvert** : après le délai, **un** appel d'essai. Réussi : on referme. Échoué : on rouvre.

Un disjoncteur protège à la fois la dépendance (qui récupère sans être assaillie) et l'appelant (qui ne
gaspille plus ses ressources à attendre).

### L'idempotence côté serveur : `Idempotency-Key`

Pour rendre un `POST` réessayable, le client génère une **clé** unique par opération (un UUID) et l'envoie
dans l'en-tête `Idempotency-Key`. Le serveur mémorise la réponse associée à la clé ; une requête qui
rejoue la même clé reçoit la **même** réponse, sans réexécution. Le cas difficile est la requête rejouée
**pendant** que la première s'exécute encore (le client a abandonné et réessayé) : la seconde doit
**attendre** la fin de la première, pas la réexécuter. Mesuré dans l'exemple : trois envois de la même clé,
dont deux simultanés, **une** exécution, trois réponses identiques.

En production, le stockage des clés est **partagé** entre instances (table en base avec contrainte
d'unicité sur la clé, N8 L4), les clés expirent (24 h), et une clé est liée au client qui l'a envoyée.

### Cloisons et mode dégradé

- **Cloison** (*bulkhead*) : limiter le nombre d'appels simultanés vers **chaque** dépendance (un
  sémaphore, N6 L11). Une dépendance lente ne peut alors immobiliser qu'une part bornée des goroutines ; les
  requêtes qui n'en ont pas besoin continuent d'être servies.
- **Mode dégradé** : décider, pour chaque dépendance, ce qu'on fait sans elle. Le catalogue est tombé :
  afficher l'ISBN au lieu du titre (leçon 7). Le service de courriel est tombé : l'emprunt réussit quand
  même, l'événement attend dans l'*outbox* (leçon 8). C'est une décision **métier**, à prendre avant la
  panne.

### Quand ne pas le faire

- **Réessayer à plusieurs niveaux** : le client réessaie 3 fois, la passerelle 3 fois, le service 3 fois
  sa base — une panne produit 27 tentatives. Réessayer à **un** niveau, de préférence le plus proche de
  l'appelant final.
- **Réessayer une opération non idempotente** sans clé.
- **Un disjoncteur devant une dépendance locale et fiable**, ou avec des seuils jamais mesurés : il
  s'ouvrira sur une simple pointe et créera sa propre panne.
- **Masquer les pannes** : réessais et mode dégradé rendent la panne invisible pour l'utilisateur, pas pour
  l'exploitation. Chaque réessai, chaque ouverture du disjoncteur doit être compté (leçon 10).

## Exemple

Les quatre mécanismes, testés de façon déterministe (aléa et horloge injectés). La fonction d'exemple de
test vérifie la sortie.

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
		if serr := sleep(ctx, b.Delay(i)); serr != nil {
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
// Fichier : resil/breaker.go
package resil

import (
	"errors"
	"sync"
	"time"
)

var ErrOpen = errors.New("disjoncteur ouvert : appel refusé sans essayer")

type State int

const (
	Closed   State = iota // normal : les appels passent
	Open                  // trop d'échecs : refus immédiat
	HalfOpen              // après le délai : un essai pour voir
)

func (s State) String() string { return [...]string{"fermé", "ouvert", "semi-ouvert"}[s] }

// Breaker s'ouvre après Threshold échecs consécutifs, reste ouvert Cooldown, puis laisse
// passer UN appel d'essai : succès → fermé, échec → ouvert de nouveau.
type Breaker struct {
	Threshold int
	Cooldown  time.Duration
	Now       func() time.Time

	mu       sync.Mutex
	state    State
	failures int
	openedAt time.Time
	probing  bool
}

func (b *Breaker) State() State {
	b.mu.Lock()
	defer b.mu.Unlock()
	return b.state
}

func (b *Breaker) Do(op func() error) error {
	b.mu.Lock()
	if b.state == Open && b.Now().Sub(b.openedAt) >= b.Cooldown {
		b.state = HalfOpen
	}
	if b.state == Open || (b.state == HalfOpen && b.probing) {
		b.mu.Unlock()
		return ErrOpen
	}
	if b.state == HalfOpen {
		b.probing = true // un seul essai à la fois
	}
	b.mu.Unlock()

	err := op()

	b.mu.Lock()
	defer b.mu.Unlock()
	b.probing = false
	if err == nil {
		b.state, b.failures = Closed, 0
		return nil
	}
	b.failures++
	if b.state == HalfOpen || b.failures >= b.Threshold {
		b.state, b.openedAt = Open, b.Now()
	}
	return err
}
```

```go
// Fichier : resil/idempotency.go
package resil

import (
	"bytes"
	"net/http"
	"net/http/httptest"
	"sync"
)

type stored struct {
	done   chan struct{} // fermé quand la première exécution est terminée
	status int
	body   []byte
}

// Idempotent : un POST rejoué avec la même clé Idempotency-Key rend la MÊME réponse,
// sans réexécuter le handler — y compris si les deux arrivent en même temps.
// (En production : stockage partagé entre instances, avec expiration, et clé liée au client.)
func Idempotent(next http.Handler) http.Handler {
	var mu sync.Mutex
	seen := map[string]*stored{}
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		key := r.Header.Get("Idempotency-Key")
		if key == "" || r.Method != http.MethodPost {
			next.ServeHTTP(w, r)
			return
		}
		mu.Lock()
		s, dup := seen[key]
		if !dup {
			s = &stored{done: make(chan struct{})}
			seen[key] = s
		}
		mu.Unlock()
		if !dup {
			rec := httptest.NewRecorder() // capturer la réponse pour la rejouer
			next.ServeHTTP(rec, r)
			s.status, s.body = rec.Code, rec.Body.Bytes()
			close(s.done)
		} else {
			select { // la première exécution est peut-être encore en cours
			case <-s.done:
			case <-r.Context().Done():
				return
			}
			w.Header().Set("Idempotent-Replayed", "true")
		}
		w.WriteHeader(s.status)
		bytes.NewReader(s.body).WriteTo(w)
	})
}
```

```go
// Fichier : resil/example_test.go
package resil_test

import (
	"context"
	"errors"
	"fmt"
	"math/rand/v2"
	"net/http"
	"net/http/httptest"
	"slices"
	"strings"
	"sync"
	"sync/atomic"
	"time"

	"exemple/resil"
)

func Example() {
	ctx := context.Background()
	rng := rand.New(rand.NewPCG(1, 2)) // aléa déterministe
	backoff := resil.Backoff{Base: 100 * time.Millisecond, Max: 2 * time.Second, Rand: rng}

	// 1. Réessais : une dépendance qui échoue deux fois, puis répond.
	calls := 0
	var waited []time.Duration
	fakeSleep := func(_ context.Context, d time.Duration) error { waited = append(waited, d); return nil }
	err := resil.Retry(ctx, 5, backoff, fakeSleep, func(context.Context) error {
		if calls++; calls < 3 {
			return errors.New("503")
		}
		return nil
	})
	fmt.Printf("réessais : %d appels, attentes %v, err=%v\n", calls, round(waited), err)

	// Une erreur permanente n'est pas réessayée.
	calls = 0
	err = resil.Retry(ctx, 5, backoff, fakeSleep, func(context.Context) error {
		calls++
		return resil.Permanent{Err: errors.New("404 : livre inconnu")}
	})
	fmt.Printf("permanente : %d appel, err=%v\n", calls, err)

	// 2. Le troupeau : 100 clients échouent au même instant et réessaient.
	for _, jitter := range []bool{false, true} {
		buckets := map[int]int{} // nombre de réessais par tranche de 10 ms
		for range 100 {
			d := 100 * time.Millisecond // sans jitter : exactement la même attente
			if jitter {
				d = backoff.Delay(1)
			}
			buckets[int(d/(10*time.Millisecond))]++
		}
		peak := slices.Max(mapsValues(buckets))
		fmt.Printf("troupeau, jitter=%v : %d tranche(s) de 10 ms occupée(s), pic %d réessais simultanés\n",
			jitter, len(buckets), peak)
	}

	// 3. Disjoncteur, avec une horloge simulée.
	now := time.Date(2026, 3, 1, 0, 0, 0, 0, time.UTC)
	br := &resil.Breaker{Threshold: 3, Cooldown: 30 * time.Second, Now: func() time.Time { return now }}
	down := errors.New("catalogue en panne")
	var trace []string
	for i := range 5 {
		err := br.Do(func() error { return down })
		trace = append(trace, fmt.Sprintf("%d:%s", i+1, short(err)))
	}
	fmt.Println("disjoncteur :", strings.Join(trace, " "), "→", br.State())
	now = now.Add(31 * time.Second)
	fmt.Println("après 31 s, essai réussi :", br.Do(func() error { return nil }), "→", br.State())

	// 4. Idempotence : le même POST, deux fois en même temps, puis une troisième fois.
	var executed atomic.Int64
	h := resil.Idempotent(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		n := executed.Add(1)
		time.Sleep(20 * time.Millisecond) // le traitement prend du temps
		w.WriteHeader(http.StatusCreated)
		fmt.Fprintf(w, "emprunt n°%d", n)
	}))
	var wg sync.WaitGroup
	bodies := make([]string, 3)
	for i := range 3 {
		if i == 2 {
			wg.Wait() // la troisième arrive après les deux premières
		}
		wg.Go(func() {
			req := httptest.NewRequest("POST", "/loans", nil)
			req.Header.Set("Idempotency-Key", "cle-123")
			rec := httptest.NewRecorder()
			h.ServeHTTP(rec, req)
			bodies[i] = fmt.Sprintf("%d %s", rec.Code, rec.Body)
		})
	}
	wg.Wait()
	fmt.Printf("idempotence : exécutions=%d, réponses=%q\n", executed.Load(), bodies)
	// Output:
	// réessais : 3 appels, attentes [77ms 123ms], err=<nil>
	// permanente : 1 appel, err=404 : livre inconnu
	// troupeau, jitter=false : 1 tranche(s) de 10 ms occupée(s), pic 100 réessais simultanés
	// troupeau, jitter=true : 10 tranche(s) de 10 ms occupée(s), pic 13 réessais simultanés
	// disjoncteur : 1:échec 2:échec 3:échec 4:refusé 5:refusé → ouvert
	// après 31 s, essai réussi : <nil> → fermé
	// idempotence : exécutions=1, réponses=["201 emprunt n°1" "201 emprunt n°1" "201 emprunt n°1"]
}

func round(ds []time.Duration) []time.Duration {
	out := make([]time.Duration, len(ds))
	for i, d := range ds {
		out[i] = d.Round(time.Millisecond)
	}
	return out
}

func mapsValues(m map[int]int) []int {
	var v []int
	for _, x := range m {
		v = append(v, x)
	}
	return v
}

func short(err error) string {
	if errors.Is(err, resil.ErrOpen) {
		return "refusé"
	}
	return "échec"
}
```
`go test -race ./...` passe.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Backoff.Delay` | Plafond exponentiel `Base × 2^(n-1)` borné à `Max`, puis tirage uniforme dans `[0, plafond]`. |
| `Rand` injecté | Le test est déterministe : mêmes attentes à chaque exécution (`77ms`, `123ms`). |
| `sleep` injecté | Le test des réessais est instantané ; en production, `resil.Sleep`, interrompu par `ctx`. |
| `Permanent` + `errors.As` | Une erreur marquée permanente arrête les réessais dès le premier appel. |
| `errors.Join(serr, err)` | Budget épuisé : l'appelant reçoit l'annulation **et** la dernière cause. |
| Simulation du troupeau | 100 réessais dans la même tranche de 10 ms sans aléa ; pic de 13 avec. |
| `Breaker` | Trois échecs ouvrent ; les appels 4 et 5 sont refusés sans essayer ; après 31 s, un essai réussi referme. |
| `probing` | En semi-ouvert, un seul essai à la fois : les autres appelants sont refusés pendant l'essai. |
| `Idempotent` | La première requête s'exécute et sa réponse est capturée ; les suivantes attendent `done` et la rejouent. |
| `Idempotent-Replayed: true` | Le client peut savoir qu'il reçoit une réponse rejouée. |
| `exécutions=1` | Trois envois de la même clé (deux simultanés) : un seul emprunt. |

## Erreurs fréquentes

1. **Pas de délai** sur les appels sortants.
2. **Réessayer les erreurs permanentes** (400, 404) ou les `POST` sans clé d'idempotence.
3. **Des réessais sans attente**, ou à intervalles fixes et synchronisés (pas de *jitter*).
4. **Des réessais à chaque niveau** de la chaîne d'appels.
5. **Un délai par tentative plus long que le budget global.**
6. **Un disjoncteur jamais refermé**, ou qui laisse passer toutes les requêtes en semi-ouvert.
7. **Une clé d'idempotence stockée en mémoire** d'une seule instance derrière un répartiteur de charge.
8. **Aucun mode dégradé décidé** : l'erreur de la dépendance remonte telle quelle à l'utilisateur.

## Bonnes pratiques Go

- `context.WithTimeout` sur chaque appel sortant ; budget par tentative < budget global.
- Réessais bornés, exponentiels, avec *full jitter* ; seulement sur erreurs passagères d'opérations
  idempotentes ; `Retry-After` respecté.
- Aléa et horloge injectés pour tester réessais et disjoncteurs sans attendre.
- Disjoncteur par dépendance, seuils mesurés ; ouverture et refus comptés en métriques.
- `Idempotency-Key` pour les `POST` réessayables, stockée de façon partagée, avec expiration.
- Une cloison (sémaphore) par dépendance lente ; un mode dégradé décidé à l'avance.

## Ce que je dois retenir

- Délai d'abord : sans lui, rien d'autre ne fonctionne.
- Réessayer = erreur passagère **et** opération idempotente ; un seul niveau de réessais.
- Exponentielle bornée + *full jitter* : sinon le troupeau des réessais refait tomber la dépendance.
- Disjoncteur : fermé, ouvert (refus immédiat), semi-ouvert (un essai).
- Clé d'idempotence : un `POST` rejoué rend la même réponse, sans réexécution.

---

➡️ [Exercices](exercices.md)
