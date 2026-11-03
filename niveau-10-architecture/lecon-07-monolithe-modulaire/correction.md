# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — La frontière `internal`

```
package exemple/lending
	lending/hack.go:3:8: use of internal package exemple/catalog/internal/store not allowed
```

Déplacé en `exemple/internal/store`, il devient importable par **tout** le module (`exemple/…`) : le parent
de ce `internal` est la racine. La protection n'existe plus entre modules. Règle : dans un monolithe
modulaire, chaque module a **son** `internal/` (`catalog/internal`, `lending/internal`) ; un
`internal/` à la racine ne sert qu'à cacher du code au monde **extérieur** au dépôt, pas aux autres modules.

## E2 — Le test d'architecture

Voir `arch_test.go` ci-dessous : il lit les imports avec `go list` et échoue si un paquet de `lending/…`
importe autre chose que `exemple/catalog` dans le catalogue, ou si le catalogue importe le prêt. Ajouter un
import de `exemple/catalog/httpapi` dans `lending` le fait échouer avec
`exemple/lending importe exemple/catalog/httpapi : seule l'API publique exemple/catalog est permise`.

## E3 — Le prix du réseau, en vrai

Avec 1 ms de délai simulé, un appel mesuré coûte ≈ 2,5 ms (le délai, plus le surcoût de `time.Sleep`, de
HTTP et de l'ordonnancement). Une goroutine produit donc ≈ 400 reçus par seconde, contre plusieurs
millions en mémoire. Avec 20 titres demandés un par un : 20 × 2,5 ms = **50 ms** par page (mesuré), soit
20 pages par seconde et par goroutine. Sur un vrai réseau entre deux machines (0,5 à 2 ms d'aller-retour,
plus en cas de chiffrement et de répartiteur de charge), c'est le même ordre de grandeur : les appels en
série par élément sont le N+1 des services (N8 L8).

## E4 — Les pannes

Sans délai, une réponse en 30 s bloque la goroutine du reçu 30 s, et toutes les requêtes HTTP qui
attendent ce reçu avec elle. Les deux délais ont des rôles différents :

- `context.WithTimeout` côté `lending` exprime le **budget** du cas d'usage (« un reçu en 500 ms au
  plus ») et se propage : il annule aussi ce qui se passe en aval ;
- le délai du `http.Client` (ou de son `Transport`) est un **filet** technique, valable pour tous les appels
  de ce client, même ceux dont l'appelant a oublié le contexte.

Il faut le premier, parce que c'est lui qui porte l'intention et qui se combine avec le délai de la requête
HTTP entrante (N7 L7) ; le second en complément.

## E5 — Les tables partagées

Conséquences de `loans JOIN works` : le catalogue ne peut plus renommer ni restructurer sa table sans
casser le prêt ; extraire le catalogue en service devient impossible sans réécrire la requête ; et les
deux modules doivent toujours partager la **même** base (et ses migrations). Alternatives : appeler l'API du
catalogue (par lot, exercice intermédiaire) ; ou garder dans le module de prêt une **copie** des titres,
tenue à jour par les événements du catalogue (leçon 8) — une lecture locale, rapide, et un couplage limité
au format de l'événement.

## Exercice intermédiaire — Un appel par lot

Les ajouts au catalogue (`Titles` et la route `GET /works?isbn=…`) :

```go
// Fichier : catalog/catalog.go
// Package catalog : l'API PUBLIQUE du module. C'est tout ce que les autres modules voient.
package catalog

import (
	"context"
	"errors"

	"exemple/catalog/internal/store"
)

var ErrUnknownWork = errors.New("œuvre inconnue")

type Module struct{ store *store.Store }

func New(seed map[string]string) *Module { return &Module{store: store.New(seed)} }

func (m *Module) Title(_ context.Context, isbn string) (string, error) {
	if t, ok := m.store.Title(isbn); ok {
		return t, nil
	}
	return "", ErrUnknownWork
}

// Titles : un appel pour plusieurs œuvres. Les ISBN inconnus sont simplement ABSENTS du
// résultat (résultat partiel, pas d'erreur) : décision documentée et testée.
func (m *Module) Titles(_ context.Context, isbns []string) (map[string]string, error) {
	out := make(map[string]string, len(isbns))
	for _, i := range isbns {
		if t, ok := m.store.Title(i); ok {
			out[i] = t
		}
	}
	return out, nil
}
```

```go
// Fichier : catalog/httpapi/httpapi.go
// Le même module, exposé en HTTP : ce qu'on écrirait le jour où on l'extrait en service.
package httpapi

import (
	"encoding/json"
	"errors"
	"net/http"
	"time"

	"exemple/catalog"
)

// Latency simule le réseau (tests et mesures uniquement).
var Latency time.Duration

func Handler(m *catalog.Module) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /works", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(Latency)
		isbns := r.URL.Query()["isbn"]
		if len(isbns) > 100 {
			http.Error(w, "100 ISBN au plus", http.StatusBadRequest)
			return
		}
		titles, _ := m.Titles(r.Context(), isbns)
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(titles)
	})
	mux.HandleFunc("GET /works/{isbn}", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(Latency)
		t, err := m.Title(r.Context(), r.PathValue("isbn"))
		if errors.Is(err, catalog.ErrUnknownWork) {
			http.Error(w, err.Error(), http.StatusNotFound)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"title": t})
	})
	return mux
}
```

```go
// Fichier : lending/batch.go
package lending

import (
	"context"
	"fmt"
	"strings"
)

type titlesSource interface {
	Titles(ctx context.Context, isbns []string) (map[string]string, error)
}

// LoanList affiche les emprunts ; un titre inconnu (ou indisponible) est remplacé par l'ISBN.
func LoanList(ctx context.Context, src titlesSource, member string, isbns []string) (string, error) {
	titles, err := src.Titles(ctx, isbns)
	if err != nil {
		return "", err
	}
	lines := make([]string, len(isbns))
	for i, isbn := range isbns {
		t, ok := titles[isbn]
		if !ok {
			t = "ISBN " + isbn
		}
		lines[i] = "- " + t
	}
	return fmt.Sprintf("%s :\n%s", member, strings.Join(lines, "\n")), nil
}
```

```go
// Fichier : lending/catalogclient/titles.go
package catalogclient

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
)

func (c Client) Titles(ctx context.Context, isbns []string) (map[string]string, error) {
	q := url.Values{"isbn": isbns}
	req, err := http.NewRequestWithContext(ctx, "GET", c.Base+"/works?"+q.Encode(), nil)
	if err != nil {
		return nil, err
	}
	resp, err := c.HTTP.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("catalogue : statut %d", resp.StatusCode)
	}
	out := map[string]string{}
	return out, json.NewDecoder(resp.Body).Decode(&out)
}
```

Mesuré, 1 ms de latence simulée : **20 appels : 50 ms ; 1 appel par lot : 3,1 ms**. En mémoire, la
différence est négligeable ; à travers le réseau, elle est de ×16. Une API **destinée** au réseau doit donc
proposer des opérations par lot dès sa conception (avec une borne : 100 ISBN au plus ici) — ce qui n'était
pas nécessaire tant que le catalogue était un module en mémoire, et c'est l'une des raisons pour
lesquelles une extraction n'est jamais un simple déplacement de code.

**ISBN inconnus** : résultat **partiel**, sans erreur ; l'affichage remplace un titre manquant par l'ISBN.
Une erreur globale pour un seul ISBN inconnu rendrait toute la page inutilisable. Le test
`TestTitlesPartialOnBothAssemblies` vérifie ce comportement sur les deux assemblages.

## Défi — Extraire le catalogue

Le client d'un catalogue extrait :

```go
// Fichier : lending/catalogclient/resilient.go
// Défi : le client d'un service EXTRAIT. Délai par appel, réessais, cache.
package catalogclient

import (
	"context"
	"errors"
	"math/rand/v2"
	"net"
	"strings"
	"sync"
	"time"
)

type Resilient struct {
	Next    Client
	Timeout time.Duration
	Retries int
	TTL     time.Duration
	Now     func() time.Time

	mu    sync.Mutex
	cache map[string]entry
}

type entry struct {
	title string
	at    time.Time
}

func (r *Resilient) Title(ctx context.Context, isbn string) (string, error) {
	r.mu.Lock()
	if e, ok := r.cache[isbn]; ok && r.Now().Sub(e.at) < r.TTL {
		r.mu.Unlock()
		return e.title, nil
	}
	r.mu.Unlock()

	var err error
	for attempt := 0; attempt <= r.Retries; attempt++ {
		if attempt > 0 { // attente aléatoire entre 10 et 20 ms × tentative (leçon 9)
			d := time.Duration(attempt) * time.Duration(10+rand.IntN(10)) * time.Millisecond
			select {
			case <-ctx.Done():
				return "", ctx.Err()
			case <-time.After(d):
			}
		}
		var t string
		actx, cancel := context.WithTimeout(ctx, r.Timeout)
		t, err = r.Next.Title(actx, isbn)
		cancel()
		if err == nil {
			r.mu.Lock()
			if r.cache == nil {
				r.cache = map[string]entry{}
			}
			r.cache[isbn] = entry{t, r.Now()}
			r.mu.Unlock()
			return t, nil
		}
		if !retryable(err) {
			return "", err
		}
	}
	return "", err
}

// retryable : erreurs réseau, délais, 503. Pas un 404 (œuvre inconnue) : réessayer ne changera rien.
func retryable(err error) bool {
	var ne net.Error
	return errors.As(err, &ne) || errors.Is(err, context.DeadlineExceeded) ||
		strings.Contains(err.Error(), "statut 503")
}
```

```go
// Fichier : lending/corr_test.go
package lending_test

import (
	"context"
	"net/http"
	"net/http/httptest"
	"strings"
	"sync/atomic"
	"testing"
	"time"

	"exemple/catalog"
	"exemple/catalog/httpapi"
	"exemple/lending"
	"exemple/lending/catalogclient"
)

var many = func() map[string]string {
	m := map[string]string{}
	for i := range 20 {
		m[string(rune('a'+i))] = "Titre " + string(rune('a'+i))
	}
	return m
}()

func keys() []string {
	var k []string
	for i := range 20 {
		k = append(k, string(rune('a'+i)))
	}
	return k
}

func TestTitlesPartialOnBothAssemblies(t *testing.T) {
	srv := httptest.NewServer(httpapi.Handler(catalog.New(many)))
	defer srv.Close()
	for name, src := range map[string]interface {
		Titles(context.Context, []string) (map[string]string, error)
	}{
		"mémoire": catalog.New(many),
		"HTTP":    catalogclient.Client{Base: srv.URL, HTTP: srv.Client()},
	} {
		got, err := lending.LoanList(t.Context(), src, "ada", []string{"a", "zz", "b"})
		want := "ada :\n- Titre a\n- ISBN zz\n- Titre b"
		if err != nil || got != want {
			t.Errorf("%s : %q %v", name, got, err)
		}
	}
}

// Mesure : 20 appels unitaires contre 1 appel par lot, avec 1 ms de « réseau ».
func TestBatchVersusOneByOne(t *testing.T) {
	httpapi.Latency = time.Millisecond
	defer func() { httpapi.Latency = 0 }()
	srv := httptest.NewServer(httpapi.Handler(catalog.New(many)))
	defer srv.Close()
	c := catalogclient.Client{Base: srv.URL, HTTP: srv.Client()}

	start := time.Now()
	for _, k := range keys() {
		if _, err := c.Title(t.Context(), k); err != nil {
			t.Fatal(err)
		}
	}
	oneByOne := time.Since(start)
	start = time.Now()
	if _, err := c.Titles(t.Context(), keys()); err != nil {
		t.Fatal(err)
	}
	batch := time.Since(start)
	t.Logf("20 appels : %v ; 1 appel par lot : %v", oneByOne.Round(time.Millisecond), batch.Round(100*time.Microsecond))
	if batch*5 > oneByOne {
		t.Errorf("le lot devrait être bien plus rapide")
	}
}

func TestResilientClient(t *testing.T) {
	var calls atomic.Int64
	var fail atomic.Int64
	fail.Store(2) // les 2 premiers appels échouent en 503
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		calls.Add(1)
		if fail.Add(-1) >= 0 {
			http.Error(w, "indisponible", http.StatusServiceUnavailable)
			return
		}
		w.Write([]byte(`{"title":"La Peste"}`))
	}))
	clock := time.Date(2026, 3, 1, 0, 0, 0, 0, time.UTC)
	r := &catalogclient.Resilient{
		Next: catalogclient.Client{Base: srv.URL, HTTP: srv.Client()}, Timeout: time.Second,
		Retries: 2, TTL: 5 * time.Minute, Now: func() time.Time { return clock },
	}
	if got, err := r.Title(t.Context(), "x"); err != nil || got != "La Peste" || calls.Load() != 3 {
		t.Fatalf("%q %v après %d appels", got, err, calls.Load())
	}
	srv.Close() // le catalogue tombe…
	if got, err := r.Title(t.Context(), "x"); err != nil || got != "La Peste" {
		t.Fatalf("le cache doit servir pendant la panne : %q %v", got, err)
	}
	clock = clock.Add(6 * time.Minute) // … et le cache expire
	_, err := r.Title(t.Context(), "x")
	if err == nil || !strings.Contains(err.Error(), "connection refused") {
		t.Fatalf("erreur réseau attendue : %v", err)
	}
	t.Log(err)
}
```

```go
// Fichier : arch_test.go
package arch

import (
	"os/exec"
	"strings"
	"testing"
)

func deps(t *testing.T, pattern string) map[string][]string {
	t.Helper()
	out, err := exec.Command("go", "list", "-f", `{{.ImportPath}} {{join .Imports " "}}`, pattern).Output()
	if err != nil {
		t.Fatal(err)
	}
	m := map[string][]string{}
	for line := range strings.Lines(string(out)) {
		f := strings.Fields(line)
		m[f[0]] = f[1:]
	}
	return m
}

func TestModuleBoundaries(t *testing.T) {
	for pkg, imps := range deps(t, "./lending/...") {
		for _, imp := range imps {
			if strings.HasPrefix(imp, "exemple/catalog") && imp != "exemple/catalog" {
				t.Errorf("%s importe %s : seule l'API publique exemple/catalog est permise", pkg, imp)
			}
		}
	}
	for pkg, imps := range deps(t, "./catalog/...") {
		for _, imp := range imps {
			if strings.HasPrefix(imp, "exemple/lending") {
				t.Errorf("%s importe %s : le catalogue ne dépend pas du prêt", pkg, imp)
			}
		}
	}
}
```

`go test -race ./...` passe. `TestResilientClient` montre les trois comportements : deux 503 puis succès
(3 appels, grâce aux réessais) ; panne totale du catalogue mais titre servi **depuis le cache** ; cache
expiré pendant la panne → erreur réseau, que l'appelant affiche en mode dégradé (« ISBN … » au lieu du
titre, comme `LoanList`).

L'assemblage (`cmd/biblio`) suit la leçon 3 : si `CATALOG_URL` est vide, `lending.New(catalog.New(…))` ;
sinon `lending.New(&catalogclient.Resilient{Next: catalogclient.Client{Base: url, HTTP: client}, …})`. Le
module de prêt ne change pas.

**Bilan de l'extraction**, en dix lignes :

1. Code ajouté : un serveur HTTP, un client, un client « résilient », un cache, des routes par lot.
2. Nouveaux modes de panne : service injoignable, lent, partiellement disponible ; cache périmé.
3. Nouvelles décisions : délais, nombre de réessais, durée du cache, comportement dégradé.
4. Exploitation : un deuxième processus à déployer, configurer, surveiller, mettre à jour.
5. Versionnement : l'API du catalogue ne peut plus changer sans compatibilité ascendante.
6. Performance : ×400 par appel ; compensée par le lot et le cache, pas annulée.
7. Gain : le catalogue peut être déployé et mis à l'échelle seul, par une autre équipe.
8. Gain : une panne du catalogue n'empêche plus les emprunts (avec le mode dégradé).
9. Pour ce module (des titres, en lecture, peu changeants), aucun de ces gains n'est indispensable.
10. Verdict : extraction **non justifiée** ici ; le module en mémoire, avec des frontières vérifiées, suffit.

## Quiz

1. Un seul programme découpé en modules aux frontières vérifiées, chacun avec son API publique et ses
   données.
2. Qu'un paquet `internal` n'est importable que sous son parent ; pas les imports entre APIs publiques, ni
   les accès aux tables.
3. Parce qu'une jointure couple les schémas et empêche toute évolution ou extraction indépendante.
4. Par un test d'architecture fondé sur `go list`, et par des droits en base.
5. Pannes partielles, perte des transactions, versionnement, observabilité distribuée, exploitation.
6. Bonnes : ressources différentes, équipe et rythme de déploiement distincts, isolation (sécurité,
   pannes) ; mauvaises : la mode, « pour que le code soit propre ».
7. Par des interfaces côté consommateur et l'absence de données partagées : l'extraction ne change que
   l'assemblage.
8. Des services qu'on doit déployer ensemble, qui partagent une base ou s'appellent en chaîne pour chaque
   requête.
9. Parce que chaque appel coûte un aller-retour réseau : N appels unitaires = N allers-retours.
10. Une transaction ne peut plus couvrir les deux côtés : cohérence à terme, événements, compensations.
