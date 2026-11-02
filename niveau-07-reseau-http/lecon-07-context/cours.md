# Leçon 7 — `context` de bout en bout

> Un utilisateur clique, puis ferme l'onglet. Sa requête a déjà atteint notre serveur, qui interroge
> une base de données et appelle un autre service. Si rien ne propage l'abandon, le serveur fait tout
> ce travail **pour personne** — et sous charge, ces requêtes fantômes saturent la base. La leçon 8 du
> niveau 6 a donné l'outil (`context`) ; celle-ci le branche **de bout en bout** : client → serveur →
> dépendances, et jusqu'à l'**arrêt gracieux** du serveur lui-même.

## Objectifs

1. Utiliser `r.Context()` : annulé quand le client part.
2. Ajouter un budget de temps par requête et le propager à toutes les dépendances.
3. Distinguer « délai dépassé » (504) de « client parti » (ne rien répondre).
4. Arrêter un serveur gracieusement avec `http.Server.Shutdown` et `signal.NotifyContext`.

## Explication

### Le contexte de la requête

Pour chaque requête, le serveur crée un contexte, accessible par `r.Context()`, qui est **annulé** :

- quand le client **ferme la connexion** (onglet fermé, délai côté client, réseau coupé) ;
- quand le handler **se termine** (`ServeHTTP` a rendu) ;
- pour une requête HTTP/2, quand le client annule ce flux.

C'est la racine de tout le travail fait pour cette requête : on ne crée **jamais** de
`context.Background()` dans un handler (N6 L8 : la chaîne serait coupée).

### Le budget et la propagation

```go
ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
defer cancel()

user, err := db.QueryRowContext(ctx, …)          // niveau 8
req, _ := http.NewRequestWithContext(ctx, …)       // leçon 6
```

Le contexte dérivé est annulé par le **premier** des deux événements : départ du client, ou fin du
budget. Toutes les dépendances le reçoivent, et toutes celles de la bibliothèque standard
(`database/sql`, `net/http`, `net`) le respectent. Le budget se **partage** : si la requête a 2 s et que
la base en a pris 1,5, l'appel HTTP suivant n'a plus que 0,5 s — sans aucun calcul de votre part.

Un service qui appelle un autre service **transmet** ainsi l'abandon d'un bout à l'autre : le client
annule → notre serveur annule son appel sortant → le serveur distant voit son propre `r.Context()`
annulé → il annule sa requête SQL. Une seule fermeture d'onglet libère des ressources sur trois machines.

### Trois issues à distinguer

| Erreur | Signification | Réponse |
|---|---|---|
| `context.DeadlineExceeded` | **notre** budget est épuisé | **504** Gateway Timeout (ou 503) — une réponse utile pour le client |
| `context.Canceled` | le client est **parti** | **rien** : personne ne lira la réponse ; éventuellement un compteur |
| autre | une vraie erreur | selon la leçon 5 |

Journaliser les annulations client en erreur noie les vraies erreurs : c'est un comportement **normal**.

### L'arrêt gracieux

Un déploiement arrête l'ancienne version du service (signal `SIGTERM`). Un `os.Exit` brutal coupe les
requêtes en cours : des clients reçoivent une erreur de connexion pour des requêtes qui auraient
réussi. L'arrêt **gracieux** :

1. cesse d'**accepter** de nouvelles connexions ;
2. ferme les connexions **inactives** (*keep-alive*) ;
3. **attend** que les requêtes en cours se terminent — dans la limite d'un délai ;
4. rend la main.

C'est exactement `srv.Shutdown(ctx)`. Le contexte passé à `Shutdown` borne l'attente : après lui,
`Shutdown` rend une erreur, et on peut fermer de force (`srv.Close()`).

Les points délicats, tous dans l'exemple :

- `ListenAndServe` **bloque** : on le lance dans une goroutine, et après `Shutdown`, il rend
  **`http.ErrServerClosed`** — ce n'est pas une erreur ;
- le contexte de `Shutdown` doit être **neuf** (`context.Background()` + délai) : le contexte de
  signal est déjà annulé à ce moment-là, et `Shutdown` rendrait immédiatement ;
- `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)` transforme les signaux en annulation de
  contexte (déjà vu au N6 L9) ;
- `Shutdown` n'interrompt **pas** les connexions détournées (WebSockets, leçon 11) : il faut les
  suivre à part (`srv.RegisterOnShutdown`).

Pour que les requêtes longues s'arrêtent elles aussi pendant l'arrêt, on peut donner aux requêtes un
contexte de base annulable (`srv.BaseContext`) — à utiliser avec discernement : l'intérêt d'un arrêt
gracieux est justement de laisser finir.

### Derrière un orchestrateur

Dans Kubernetes (niveau 12), l'ordre compte : le service doit d'abord être **retiré** de la répartition
de charge, sinon de nouvelles requêtes arrivent pendant l'arrêt. D'où l'habitude de faire échouer la
sonde de disponibilité (*readiness*), d'attendre quelques secondes, **puis** d'appeler `Shutdown`.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : report/report.go
// Package report sert un rapport lent qui interroge une « base » et un service distant.
package report

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"time"
)

// Store simule une base de données qui RESPECTE son contexte (comme database/sql, niveau 8).
type Store struct {
	Delay    time.Duration
	Canceled chan error // reçoit l'erreur quand une requête est interrompue (pour l'observer en test)
}

func (s *Store) Sales(ctx context.Context) (int, error) {
	select {
	case <-time.After(s.Delay):
		return 1234, nil
	case <-ctx.Done():
		if s.Canceled != nil {
			s.Canceled <- context.Cause(ctx)
		}
		return 0, fmt.Errorf("requête ventes : %w", ctx.Err())
	}
}

type Handler struct {
	Store   *Store
	Timeout time.Duration // budget total d'une requête
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Le contexte de la requête est annulé si le client se déconnecte ;
	// on lui ajoute notre propre budget. Les deux s'appliquent : le premier qui expire gagne.
	ctx, cancel := context.WithTimeout(r.Context(), h.Timeout)
	defer cancel()

	n, err := h.Store.Sales(ctx) // propagation : la « base » reçoit le même contexte
	switch {
	case errors.Is(err, context.DeadlineExceeded):
		http.Error(w, "délai dépassé", http.StatusGatewayTimeout)
		return
	case errors.Is(err, context.Canceled):
		return // le client est parti : personne ne lira la réponse
	case err != nil:
		http.Error(w, "erreur interne", http.StatusInternalServerError)
		return
	}
	fmt.Fprintf(w, "ventes : %d\n", n)
}

// Run démarre le serveur et l'arrête GRACIEUSEMENT quand ctx est annulé :
// plus de nouvelles connexions, les requêtes en cours ont `grace` pour finir.
func Run(ctx context.Context, srv *http.Server, grace time.Duration) error {
	errc := make(chan error, 1)
	go func() { errc <- srv.ListenAndServe() }()

	select {
	case err := <-errc: // échec au démarrage (port occupé…)
		return err
	case <-ctx.Done():
	}
	shutdownCtx, cancel := context.WithTimeout(context.Background(), grace) // PAS ctx : il est déjà annulé
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("arrêt : %w", err)
	}
	if err := <-errc; !errors.Is(err, http.ErrServerClosed) { // ListenAndServe rend ErrServerClosed après Shutdown
		return err
	}
	return nil
}
```

```go
// Fichier : cmd/report/main.go
// Commande report : Ctrl-C déclenche un arrêt gracieux.
package main

import (
	"context"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"exemple/report"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	srv := &http.Server{
		Addr:              "127.0.0.1:8080",
		Handler:           &report.Handler{Store: &report.Store{Delay: 2 * time.Second}, Timeout: 5 * time.Second},
		ReadHeaderTimeout: 5 * time.Second, // leçon 9
	}
	if err := report.Run(ctx, srv, 10*time.Second); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println("arrêt propre")
}
```

```go
// Fichier : report/report_test.go
package report

import (
	"context"
	"errors"
	"io"
	"net"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"
)

// Le client abandonne : l'annulation doit atteindre la « base ».
func TestClientDisconnectCancelsStore(t *testing.T) {
	store := &Store{Delay: 10 * time.Second, Canceled: make(chan error, 1)}
	srv := httptest.NewServer(&Handler{Store: store, Timeout: time.Minute})
	defer srv.Close()

	ctx, cancel := context.WithTimeout(t.Context(), 100*time.Millisecond)
	defer cancel()
	req, _ := http.NewRequestWithContext(ctx, "GET", srv.URL, nil)
	if _, err := http.DefaultClient.Do(req); !errors.Is(err, context.DeadlineExceeded) {
		t.Fatalf("client : %v", err)
	}
	select {
	case cause := <-store.Canceled:
		if !errors.Is(cause, context.Canceled) {
			t.Errorf("cause côté base = %v, attendu context.Canceled", cause)
		}
	case <-time.After(2 * time.Second):
		t.Fatal("la base n'a jamais vu l'annulation : le contexte n'est pas propagé")
	}
}

// Le budget du serveur expire avant la base : 504.
func TestServerBudget(t *testing.T) {
	h := &Handler{Store: &Store{Delay: time.Second}, Timeout: 50 * time.Millisecond}
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))
	if rec.Code != http.StatusGatewayTimeout {
		t.Errorf("statut = %d, attendu 504", rec.Code)
	}
}

// L'arrêt gracieux laisse finir la requête en cours.
func TestGracefulShutdown(t *testing.T) {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		t.Fatal(err)
	}
	addr := ln.Addr().String()
	ln.Close() // on récupère juste un port libre pour srv.Addr
	srv := &http.Server{Addr: addr, Handler: &Handler{Store: &Store{Delay: 300 * time.Millisecond}, Timeout: time.Minute}}

	ctx, stop := context.WithCancel(t.Context())
	done := make(chan error, 1)
	go func() { done <- Run(ctx, srv, 5*time.Second) }()
	time.Sleep(50 * time.Millisecond) // laisser le serveur démarrer (test d'intégration : attente tolérée)

	type result struct {
		body string
		err  error
	}
	res := make(chan result, 1)
	go func() {
		resp, err := http.Get("http://" + addr)
		if err != nil {
			res <- result{err: err}
			return
		}
		defer resp.Body.Close()
		b, _ := io.ReadAll(resp.Body)
		res <- result{body: string(b)}
	}()
	time.Sleep(100 * time.Millisecond) // la requête est en cours (300 ms)
	stop()                             // ← signal d'arrêt pendant la requête

	r := <-res
	if r.err != nil || r.body != "ventes : 1234\n" {
		t.Errorf("requête en cours : %q, %v — elle aurait dû se terminer normalement", r.body, r.err)
	}
	if err := <-done; err != nil {
		t.Errorf("Run : %v", err)
	}
	if _, err := http.Get("http://" + addr); err == nil {
		t.Error("le serveur accepte encore des requêtes après l'arrêt")
	}
}
```

`go test -race ./...` passe (vérifié trois fois de suite). Pour l'essayer à la main :
`go run ./cmd/report`, puis `curl localhost:8080` et Ctrl-C pendant les 2 secondes de la requête :
la réponse arrive quand même, puis le programme affiche « arrêt propre ».

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `context.WithTimeout(r.Context(), h.Timeout)` | Le budget s'ajoute à l'annulation par le client. |
| `h.Store.Sales(ctx)` | La dépendance reçoit le **même** contexte : elle s'arrête dès que la requête n'a plus de sens. |
| `DeadlineExceeded` → 504, `Canceled` → rien | Deux causes, deux traitements. |
| `TestClientDisconnectCancelsStore` | Le client abandonne au bout de 100 ms ; le test vérifie que la « base » voit `context.Canceled`. Sans propagation, elle travaillerait 10 s pour rien. |
| `TestServerBudget` | Notre propre budget expire avant la base : 504. |
| `Run` : `ListenAndServe` dans une goroutine | Le `select` attend soit un échec de démarrage, soit le signal d'arrêt. |
| `context.Background()` pour `Shutdown` | Le contexte de signal est déjà annulé : il ferait échouer `Shutdown` immédiatement. |
| `ErrServerClosed` filtré | C'est la sortie **normale** de `ListenAndServe` après `Shutdown`. |
| `TestGracefulShutdown` | Arrêt demandé **pendant** une requête de 300 ms : elle se termine normalement, puis le serveur refuse les connexions. Les `time.Sleep` sont tolérés ici : c'est un test d'intégration avec un vrai réseau, que `synctest` ne sait pas virtualiser (leçon 2). |
| `net.Listen("…:0")` | Le système choisit un port libre : pas de conflit entre tests. |

## Erreurs fréquentes

1. **`context.Background()` dans un handler** : l'abandon du client n'est plus propagé.
2. **Ne pas passer le contexte** aux appels base de données et HTTP.
3. **Traiter `Canceled` comme une erreur serveur** (journal d'erreurs, 500).
4. **`os.Exit` ou `log.Fatal` sur signal** au lieu de `Shutdown`.
5. **Passer le contexte de signal (déjà annulé) à `Shutdown`.**
6. **Prendre `ErrServerClosed` pour une erreur.**
7. **Pas de délai sur `Shutdown`** : une requête bloquée empêche l'arrêt pour toujours.
8. **Oublier les goroutines de fond** (purge, workers) : elles doivent aussi s'arrêter avec le contexte.

## Bonnes pratiques Go

- `r.Context()` comme racine, un budget par requête avec `WithTimeout`, propagé partout.
- `504` pour un budget dépassé, silence pour un client parti.
- `signal.NotifyContext` + `Shutdown` avec un délai neuf.
- Toutes les goroutines de longue durée du service arrêtées par le même contexte, attendues avant de sortir.
- Un test qui vérifie la propagation de l'annulation jusqu'aux dépendances.

## Ce que je dois retenir

- `r.Context()` est annulé quand le client part ou que le handler finit.
- Dériver un budget, le passer à toutes les dépendances : l'abandon traverse tout le système.
- `DeadlineExceeded` → 504 ; `Canceled` → client parti, ne rien répondre.
- Arrêt gracieux : `Shutdown(ctxNeufAvecDélai)` ; `ListenAndServe` rend alors `ErrServerClosed`.

---

➡️ [Exercices](exercices.md)
