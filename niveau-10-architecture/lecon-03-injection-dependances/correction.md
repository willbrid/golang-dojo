# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — L'`init` qui ouvre la base

Sans `DATABASE_URL`, l'`init` ne peut rien faire de propre : il panique ou appelle `log.Fatal` **avant
même `main`**, sans possibilité de rendre une erreur. Et il s'exécute **à chaque `go test`** du paquet
(un `init` s'exécute dès que le paquet est chargé) : les tests échouent ou ouvrent une vraie connexion
avant leur première ligne. `Example_run` ne peut plus tester trois démarrages : le stockage est ouvert une
fois, globalement, par l'`init`, avec l'environnement réel. On a perdu à la fois la testabilité et le
contrôle de l'ordre.

## E2 — `log.Fatal` au milieu

Mesuré, avec un seul argument :

```
stockage ouvert : x
2026/11/03 07:48:19 usage : emprunt <adhérent> <livre>
exit status 1
```

« stockage fermé » n'apparaît **pas** : `log.Fatal` appelle `os.Exit`, qui termine le processus sans
exécuter les `defer`. Avec une vraie base, la connexion est coupée brutalement (pas grave) ; avec un
fichier en cours d'écriture par un `bufio.Writer`, les dernières données ne sont jamais écrites ; avec une
transaction, elle est abandonnée côté serveur. D'où la règle : rendre l'erreur, et ne sortir que dans `main`.

## E3 — Une option de plus

`WithClock` s'ajoute sans modifier **aucun** appel existant : les appelants qui ne l'utilisent pas ne
changent pas. Avec une struct de configuration, pareil (un champ de plus, valeur zéro par défaut). Avec
des paramètres positionnels, **tous** les appels cassent. Options et struct sont donc les deux formes
extensibles ; la différence se joue sur la lisibilité et sur le problème de E4.

## E4 — Le délai à zéro

Avec `NotifierConfig{Timeout: 0}`, le zéro est ambigu. Deux solutions :

- **un pointeur** ou `sql.Null`-like : `Timeout *time.Duration` (nil = défaut, `&0` = pas de délai) —
  précis mais lourd à écrire pour l'appelant ;
- **une valeur sentinelle documentée** : `0` = défaut, `-1` (ou une constante `NoTimeout`) = pas de délai.
  Plus simple, à condition de le documenter et de le tester.

Les options fonctionnelles évitent la question : `WithTimeout(0)` est un appel explicite, distinct de
l'absence d'appel. C'est l'un de leurs vrais avantages.

## E5 — Le graphe

```
getenv ──► Config ──► Store ─────────┐
                     Notifier(out) ──┼──► Service ──► run
time.Now ────────────────────────────┘
```

Construction : configuration, stockage, notificateur, service. Fermeture : seul le stockage a une
ressource, fermé en dernier par `defer`. Un `Cache` entre le service et le stockage : **une** ligne dans
`run` (`cache := NewCache(store)`, puis `NewService(cache, …)`) — à condition que le service dépende d'une
interface que le cache et le stockage satisfont tous deux (leçon 4). Ni le service ni le stockage ne
changent.

## Exercice intermédiaire — Un vrai `main` de service

```go
// Fichier : cmd/biblio/main.go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"net"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"time"
)

type Config struct {
	Addr            string
	Store           string
	ShutdownTimeout time.Duration
}

func loadConfig(getenv func(string) string) (Config, error) {
	c := Config{Addr: getenv("ADDR"), Store: getenv("STORE"), ShutdownTimeout: 5 * time.Second}
	if c.Addr == "" {
		c.Addr = "127.0.0.1:8080"
	}
	if c.Store == "" {
		c.Store = "memory"
	}
	var errs []error
	if c.Store != "memory" && c.Store != "postgres" {
		errs = append(errs, fmt.Errorf("STORE=%q : memory ou postgres", c.Store))
	}
	if c.Store == "postgres" && getenv("DATABASE_URL") == "" {
		errs = append(errs, errors.New("DATABASE_URL obligatoire avec STORE=postgres"))
	}
	if s := getenv("SHUTDOWN_TIMEOUT"); s != "" {
		d, err := time.ParseDuration(s)
		if err != nil || d <= 0 {
			errs = append(errs, fmt.Errorf("SHUTDOWN_TIMEOUT=%q : durée positive attendue", s))
		}
		c.ShutdownTimeout = d
	}
	return c, errors.Join(errs...)
}

// memStore : le stockage, avec une fermeture observable.
type memStore struct {
	out    io.Writer
	closed bool
}

func (s *memStore) Close() error { s.closed = true; fmt.Fprintln(s.out, "stockage fermé"); return nil }

// Deps : ce que les tests peuvent fournir à la place du monde réel.
type Deps struct {
	Getenv   func(string) string
	Stdout   io.Writer
	Listener net.Listener // nil : run écoute lui-même sur cfg.Addr
	Slow     time.Duration
}

func run(ctx context.Context, d Deps) error {
	cfg, err := loadConfig(d.Getenv)
	if err != nil {
		return fmt.Errorf("configuration : %w", err)
	}
	store := &memStore{out: d.Stdout}
	defer store.Close() // en DERNIER : après l'arrêt du serveur qui l'utilise

	mux := http.NewServeMux()
	mux.HandleFunc("GET /slow", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(d.Slow) // une requête longue en cours pendant l'arrêt
		if store.closed {
			http.Error(w, "stockage déjà fermé !", 500)
			return
		}
		fmt.Fprintln(w, "terminé")
	})
	srv := &http.Server{Handler: mux, ReadHeaderTimeout: 5 * time.Second}

	ln := d.Listener
	if ln == nil {
		if ln, err = net.Listen("tcp", cfg.Addr); err != nil {
			return err
		}
	}
	fmt.Fprintln(d.Stdout, "écoute sur", strings.Split(ln.Addr().String(), ":")[0])
	errc := make(chan error, 1)
	go func() { errc <- srv.Serve(ln) }()

	select {
	case err := <-errc:
		return err
	case <-ctx.Done():
	}
	sctx, cancel := context.WithTimeout(context.WithoutCancel(ctx), cfg.ShutdownTimeout)
	defer cancel()
	if err := srv.Shutdown(sctx); err != nil { // attend les requêtes en cours
		return fmt.Errorf("arrêt : %w", err)
	}
	fmt.Fprintln(d.Stdout, "serveur arrêté")
	return nil
}

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	if err := run(ctx, Deps{Getenv: os.Getenv, Stdout: os.Stdout}); err != nil {
		fmt.Fprintln(os.Stderr, "erreur :", err)
		os.Exit(1)
	}
}
```

```go
// Fichier : cmd/biblio/main_test.go
package main

import (
	"bytes"
	"context"
	"io"
	"net"
	"net/http"
	"strings"
	"sync"
	"testing"
	"time"
)

func env(m map[string]string) func(string) string { return func(k string) string { return m[k] } }

func TestConfigErrors(t *testing.T) {
	_, err := loadConfig(env(map[string]string{"STORE": "postgres", "SHUTDOWN_TIMEOUT": "-1s"}))
	for _, want := range []string{"DATABASE_URL obligatoire", "SHUTDOWN_TIMEOUT"} {
		if err == nil || !strings.Contains(err.Error(), want) {
			t.Errorf("attendu %q dans %v", want, err)
		}
	}
	err = run(t.Context(), Deps{Getenv: env(map[string]string{"STORE": "redis"}), Stdout: io.Discard})
	if err == nil || !strings.Contains(err.Error(), "configuration") {
		t.Fatalf("run doit refuser de démarrer : %v", err)
	}
}

type syncBuf struct {
	mu sync.Mutex
	b  bytes.Buffer
}

func (s *syncBuf) Write(p []byte) (int, error) { s.mu.Lock(); defer s.mu.Unlock(); return s.b.Write(p) }
func (s *syncBuf) String() string               { s.mu.Lock(); defer s.mu.Unlock(); return s.b.String() }

func TestGracefulShutdownOrder(t *testing.T) {
	ln, err := net.Listen("tcp", "127.0.0.1:0") // le test choisit le port et le connaît
	if err != nil {
		t.Fatal(err)
	}
	var out syncBuf
	ctx, cancel := context.WithCancel(t.Context())
	done := make(chan error, 1)
	go func() {
		done <- run(ctx, Deps{Getenv: env(nil), Stdout: &out, Listener: ln, Slow: 300 * time.Millisecond})
	}()

	resp := make(chan string, 1)
	go func() {
		r, err := http.Get("http://" + ln.Addr().String() + "/slow")
		if err != nil {
			resp <- err.Error()
			return
		}
		b, _ := io.ReadAll(r.Body)
		r.Body.Close()
		resp <- strings.TrimSpace(string(b))
	}()
	time.Sleep(100 * time.Millisecond) // la requête est en cours
	cancel()                            // « SIGTERM »

	if got := <-resp; got != "terminé" {
		t.Fatalf("requête en cours : %q", got)
	}
	if err := <-done; err != nil {
		t.Fatal(err)
	}
	if want := "écoute sur 127.0.0.1\nserveur arrêté\nstockage fermé\n"; out.String() != want {
		t.Fatalf("ordre :\n%s", out.String())
	}
}
```

`go test -race ./...` passe.

- **Le port du test.** Le test crée lui-même l'écouteur sur `127.0.0.1:0` et le passe à `run` via `Deps` :
  il connaît l'adresse avant le démarrage, sans attendre ni deviner. En production, `Listener` est nil et
  `run` écoute sur `cfg.Addr`. C'est une dépendance comme une autre (le monde extérieur : le réseau), pas un
  artifice de test. L'alternative `ready func(addr string)` fonctionne aussi, mais oblige le test à se
  synchroniser sur un rappel.
- **L'ordre d'arrêt** est vérifié : la requête lente lancée avant l'annulation se termine (`terminé`),
  puis `serveur arrêté`, puis `stockage fermé`. Si l'on fermait le stockage avant `Shutdown`, la requête
  en cours rencontrerait un stockage fermé (le handler le détecte et répond 500, et le test échoue).
- **`context.WithoutCancel(ctx)`** pour le délai d'arrêt : le `ctx` reçu est déjà annulé (c'est ce qui
  déclenche l'arrêt) ; un délai dérivé de lui serait expiré d'avance.
- **Configuration** : toutes les erreurs ensemble, et `run` refuse de démarrer **avant** d'ouvrir quoi que
  ce soit.

## Défi — Construire sans se tromper d'ordre

```go
// Fichier : closers/closers.go
// Package closers : fermer dans l'ordre inverse de l'ouverture, même en cas d'échec au milieu.
package closers

import (
	"errors"
	"fmt"
)

type Closers struct {
	fns []namedClose
}

type namedClose struct {
	name string
	fn   func() error
}

func (c *Closers) Add(name string, fn func() error) { c.fns = append(c.fns, namedClose{name, fn}) }

// Close ferme tout, du dernier ajouté au premier, et rassemble les erreurs.
func (c *Closers) Close() error {
	var errs []error
	for i := len(c.fns) - 1; i >= 0; i-- {
		if err := c.fns[i].fn(); err != nil {
			errs = append(errs, fmt.Errorf("fermeture de %s : %w", c.fns[i].name, err))
		}
	}
	c.fns = nil
	return errors.Join(errs...)
}
```

```go
// Fichier : closers/closers_test.go
package closers

import (
	"errors"
	"fmt"
	"strings"
	"testing"
)

type res struct {
	name string
	log  *[]string
	fail bool
}

func (r res) Close() error {
	*r.log = append(*r.log, r.name)
	if r.fail {
		return errors.New("refus")
	}
	return nil
}

// build ouvre quatre ressources ; failAt simule l'échec de l'ouverture n° failAt.
func build(log *[]string, failAt int) (c *Closers, err error) {
	c = &Closers{}
	defer func() {
		if err != nil { // construction ratée : fermer ce qui a été ouvert
			err = errors.Join(err, c.Close())
		}
	}()
	for i, name := range []string{"base", "client", "cache", "serveur"} {
		if i == failAt {
			return c, fmt.Errorf("ouverture de %s impossible", name)
		}
		r := res{name: name, log: log, fail: name == "client"}
		c.Add(name, r.Close)
	}
	return c, nil
}

func TestReverseOrderAndJoin(t *testing.T) {
	var log []string
	c, err := build(&log, -1)
	if err != nil {
		t.Fatal(err)
	}
	err = c.Close()
	if strings.Join(log, ",") != "serveur,cache,client,base" {
		t.Fatalf("ordre : %v", log)
	}
	if err == nil || !strings.Contains(err.Error(), "fermeture de client : refus") {
		t.Fatalf("err = %v", err)
	}
}

func TestFailureMidwayClosesOpened(t *testing.T) {
	var log []string
	_, err := build(&log, 2) // « cache » échoue
	if strings.Join(log, ",") != "client,base" {
		t.Fatalf("fermés : %v", log)
	}
	if err == nil || !strings.Contains(err.Error(), "ouverture de cache") {
		t.Fatalf("err = %v", err)
	}
	t.Log(err)
}
```

- Fermeture dans l'ordre inverse : `serveur,cache,client,base`. L'échec de fermeture du client n'empêche
  pas la fermeture de la base ; l'erreur est rapportée.
- Échec de construction au milieu (le cache) : le `defer` de `build` ferme ce qui était ouvert (`client`,
  `base`), et l'erreur rendue contient la cause **et** l'erreur de fermeture :
  `ouverture de cache impossible` / `fermeture de client : refus`.
- **`Closers` contre des `defer`** : dans une seule fonction (`run`), une suite de `defer` fait la même
  chose, plus simplement, et c'est le bon choix. `Closers` sert quand la construction est **répartie**
  (une fonction `build` qui rend le graphe à son appelant, lequel le fermera plus tard), quand il faut
  **collecter** les erreurs de fermeture, ou quand le nombre de ressources est dynamique.
- **`wire`** résout l'ordre de **construction** (il calcule l'ordre à partir des signatures des
  constructeurs) et génère aussi le nettoyage en cas d'échec (ses fournisseurs peuvent rendre une
  fonction de nettoyage, appelée dans l'ordre inverse). C'est exactement le problème de ce défi — mais
  pour une dizaine de composants, le code écrit à la main reste plus lisible que la génération.

## Quiz

1. Chaque composant reçoit ses dépendances par son constructeur ; `main` construit le graphe.
2. L'endroit unique qui lit la configuration, ouvre les ressources et assemble les composants : `main` (ou
   `run`).
3. Ils cachent les dépendances, s'exécutent dans un ordre non maîtrisé (y compris en test) et ne peuvent
   pas rendre d'erreur.
4. Tout ce que `main` lirait du monde extérieur (arguments, environnement, sorties, horloge, contexte),
   pour être testable.
5. Pour que les `defer` s'exécutent et que l'arrêt soit propre.
6. Options : réglages facultatifs, appels lisibles, zéro sans ambiguïté ; struct : réglages issus d'un
   fichier de configuration, nombreux.
7. Dans l'ordre inverse de l'ouverture ; par `defer` juste après chaque ouverture.
8. Que le composant a trop de responsabilités.
9. Il génère le code d'assemblage à partir des constructeurs ; pour la plupart des services, le code écrit
   à la main est court et plus lisible.
10. Injecter ce qui touche le monde extérieur ou doit être remplacé en test ; appeler directement les
    fonctions pures.
