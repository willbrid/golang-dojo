# Leçon 10 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le code complète le paquet `obs` du cours : `http.go` est donné en entier (identifiant assaini), les
autres fichiers sont nouveaux.

```go
// Fichier : obs/http.go
package obs

import (
	"context"
	"crypto/rand"
	"log"
	"net/http"
	"strconv"
	"time"
)

type ctxKey struct{}

// RequestID rend l'identifiant de la requête en cours (vide s'il n'y en a pas).
func RequestID(ctx context.Context) string {
	id, _ := ctx.Value(ctxKey{}).(string)
	return id
}

type statusWriter struct {
	http.ResponseWriter
	status int
}

func (s *statusWriter) WriteHeader(code int) { s.status = code; s.ResponseWriter.WriteHeader(code) }

// Middleware : à la FRONTIÈRE du service, une seule fois pour toutes les routes :
//   - identifiant de requête (repris de l'appelant, ou créé), propagé par le contexte ;
//   - les métriques RED : taux (compteur), erreurs (étiquette status), durée (histogramme) ;
//   - UNE ligne de journal par requête.
//
// L'étiquette de route est le MOTIF (« GET /books/{id} »), jamais le chemin brut.
func Middleware(next http.Handler, reg *Registry, logger *log.Logger, now func() time.Time) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get("X-Request-ID")
		if !validID(id) { // absent, trop long, ou caractères qui pourraient falsifier le journal
			id = rand.Text()[:16]
		}
		r = r.WithContext(context.WithValue(r.Context(), ctxKey{}, id))
		w.Header().Set("X-Request-ID", id)
		sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}
		start := now()
		next.ServeHTTP(sw, r)
		d := now().Sub(start)

		route := r.Pattern // rempli par le ServeMux pendant ServeHTTP
		if route == "" {
			route = "aucune" // 404 du routeur : une seule série, quel que soit le chemin
		}
		status := strconv.Itoa(sw.status)
		reg.Inc("http_requests_total", "route", route, "status", status)
		reg.Observe("http_request_duration", d, "route", route)
		logger.Printf("requête id=%s route=%q status=%s durée=%v", id, route, status, d)
	})
}

// validID : 1 à 64 caractères parmi lettres, chiffres et « - » (E3).
func validID(id string) bool {
	if id == "" || len(id) > 64 {
		return false
	}
	for _, c := range []byte(id) {
		if !(c >= 'a' && c <= 'z' || c >= 'A' && c <= 'Z' || c >= '0' && c <= '9' || c == '-') {
			return false
		}
	}
	return true
}

// Transport propage l'identifiant de requête aux appels SORTANTS : la même requête se
// retrouve dans les journaux de chaque service traversé.
type Transport struct{ Base http.RoundTripper }

func (t Transport) RoundTrip(r *http.Request) (*http.Response, error) {
	if id := RequestID(r.Context()); id != "" {
		r = r.Clone(r.Context()) // un RoundTripper ne modifie pas la requête reçue
		r.Header.Set("X-Request-ID", id)
	}
	return t.Base.RoundTrip(r)
}
```

```go
// Fichier : obs/outgoing.go
// E5 : RED pour les appels SORTANTS, étiqueté par dépendance (pas par URL : cardinalité).
package obs

import (
	"net/http"
	"strconv"
	"time"
)

type Outgoing struct {
	Base       http.RoundTripper
	Dependency string
	Reg        *Registry
	Now        func() time.Time
}

func (o Outgoing) RoundTrip(r *http.Request) (*http.Response, error) {
	start := o.Now()
	resp, err := o.Base.RoundTrip(r)
	status := "error"
	if err == nil {
		status = strconv.Itoa(resp.StatusCode)
	}
	o.Reg.Inc("client_requests_total", "dependency", o.Dependency, "method", r.Method, "status", status)
	o.Reg.Observe("client_request_duration", o.Now().Sub(start), "dependency", o.Dependency)
	return resp, err
}
```

```go
// Fichier : obs/corr_test.go
package obs_test

import (
	"bytes"
	"errors"
	"fmt"
	"log"
	"net/http"
	"net/http/httptest"
	"slices"
	"strings"
	"testing"
	"time"

	"exemple/obs"
)

func TestE1MeanLies(t *testing.T) {
	reg := obs.NewRegistry()
	var ds []time.Duration
	for range 99 {
		reg.Observe("d", 10*time.Millisecond)
		ds = append(ds, 10*time.Millisecond)
	}
	reg.Observe("d", 5*time.Second)
	ds = append(ds, 5*time.Second)
	slices.Sort(ds)
	var sum time.Duration
	for _, d := range ds {
		sum += d
	}
	var b bytes.Buffer
	reg.WriteText(&b)
	t.Logf("moyenne=%v médiane=%v max=%v\n%s", sum/100, ds[49], ds[99], b.String())
}

func TestE3HostileID(t *testing.T) {
	var logs bytes.Buffer
	h := obs.Middleware(http.NotFoundHandler(), obs.NewRegistry(), log.New(&logs, "", 0), time.Now)
	for _, id := range []string{strings.Repeat("x", 10000), "abc\nrequête id=faux route=\"/admin\" status=200"} {
		req := httptest.NewRequest("GET", "/", nil)
		req.Header.Set("X-Request-ID", id)
		rec := httptest.NewRecorder()
		h.ServeHTTP(rec, req)
		if got := rec.Header().Get("X-Request-ID"); got == id || len(got) > 64 {
			t.Fatalf("identifiant hostile repris tel quel")
		}
	}
	if n := strings.Count(logs.String(), "\n"); n != 2 {
		t.Fatalf("%d lignes de journal pour 2 requêtes : ligne injectée ?\n%s", n, logs.String())
	}
}

func TestE4Cardinality(t *testing.T) {
	wrong, right := obs.NewRegistry(), obs.NewRegistry()
	for i := range 1000 {
		err := fmt.Errorf("adhérent %d introuvable", i)
		wrong.Inc("errors_total", "error", err.Error())
		right.Inc("errors_total", "reason", "member_not_found") // valeur d'une liste FERMÉE
	}
	t.Logf("séries : message d'erreur=%d ; raison=%d", wrong.Series(), right.Series())
}

func TestE5Outgoing(t *testing.T) {
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if strings.HasSuffix(r.URL.Path, "/zz") {
			w.WriteHeader(404)
		}
	}))
	reg := obs.NewRegistry()
	c := &http.Client{Transport: obs.Outgoing{Base: http.DefaultTransport, Dependency: "catalog", Reg: reg, Now: time.Now}}
	for _, p := range []string{"/works/1", "/works/2", "/works/zz"} {
		resp, _ := c.Get(srv.URL + p)
		resp.Body.Close()
	}
	srv.Close()
	c.Get(srv.URL + "/works/3") // serveur arrêté
	var b bytes.Buffer
	reg.WriteText(&b)
	out := b.String()
	for _, want := range []string{`status="200"} 2`, `status="404"} 1`, `status="error"} 1`} {
		if !strings.Contains(out, want) {
			t.Errorf("%q absent de :\n%s", want, out)
		}
	}
	if strings.Contains(out, "/works/") {
		t.Error("l'URL ne doit pas être une étiquette")
	}
}

// Intermédiaire : métriques métier et journal sans secret.
var (
	errTooMany = errors.New("trop d'emprunts")
	errLate    = errors.New("emprunt en retard")
)

// reason : la liste FERMÉE des raisons de refus, dérivée des erreurs du domaine.
func reason(err error) string {
	switch {
	case errors.Is(err, errTooMany):
		return "too_many_loans"
	case errors.Is(err, errLate):
		return "overdue"
	default:
		return "other"
	}
}

func TestBusinessMetricsAndSafeLogs(t *testing.T) {
	reg := obs.NewRegistry()
	var logs bytes.Buffer
	logger := log.New(&logs, "", 0)
	const dsn = "postgres://app:S3cret@db/biblio"
	borrow := func(member string) error {
		switch member {
		case "bob":
			return fmt.Errorf("adhérent %s : %w", member, errTooMany)
		case "zoe":
			return fmt.Errorf("adhérent %s : %w", member, errLate)
		case "panne":
			return fmt.Errorf("connexion à la base impossible") // le DSN n'est JAMAIS dans l'erreur
		}
		return nil
	}
	mux := http.NewServeMux()
	mux.HandleFunc("POST /members/{id}/loans", func(w http.ResponseWriter, r *http.Request) {
		err := borrow(r.PathValue("id"))
		switch {
		case err == nil:
			reg.Inc("loans_total")
			w.WriteHeader(201)
		case reason(err) != "other":
			reg.Inc("loans_refused_total", "reason", reason(err))
			w.WriteHeader(409)
		default:
			logger.Printf("erreur id=%s cause=%q", obs.RequestID(r.Context()), err)
			w.WriteHeader(500)
		}
	})
	h := obs.Middleware(mux, reg, logger, time.Now)
	for _, m := range []string{"ada", "bob", "zoe", "bob", "panne"} {
		req := httptest.NewRequest("POST", "/members/"+m+"/loans", nil)
		req.Header.Set("X-Request-ID", "req-"+m)
		h.ServeHTTP(httptest.NewRecorder(), req)
	}
	var b bytes.Buffer
	reg.WriteText(&b)
	t.Logf("\n%s%s", b.String(), logs.String())
	for _, want := range []string{"loans_total 1", `reason="too_many_loans"} 2`, `reason="overdue"} 1`, "erreur id=req-panne"} {
		if !strings.Contains(b.String()+logs.String(), want) {
			t.Errorf("%q absent", want)
		}
	}
	if strings.Contains(logs.String(), "S3cret") || strings.Contains(logs.String(), dsn) {
		t.Fatal("secret dans les journaux")
	}
}
```

## E1 — La moyenne qui ment

```
moyenne=59.9ms médiane=10ms max=5s
d n=100 moyenne=59.9ms [≤5ms:0 ≤25ms:99 ≤100ms:0 ≤500ms:0 ≤+Inf:1]
```

La moyenne (60 ms) ne correspond à **aucune** requête réelle : 99 ont pris 10 ms, une a pris 5 s. La
médiane dit « la plupart vont bien » ; seul l'histogramme montre la requête à 5 s, celle de l'utilisateur
qui va se plaindre. Le p99 (ici, la durée de la 99ᵉ requête la plus rapide sur 100) et le maximum sont les
valeurs qui reflètent l'expérience des malchanceux.

## E2 — Exposer `/metrics`

Si `/metrics` passe par le middleware, chaque lecture du collecteur (toutes les 15 s, typiquement)
apparaît dans les métriques de trafic et les fausse légèrement ; ce n'est pas grave, mais une route
d'administration se range mieux à part. Surtout, `/metrics` ne doit **pas** être accessible depuis
Internet : elle révèle les routes, les volumes, les erreurs. Solution : un **second** `http.Server` sur une
adresse interne (`127.0.0.1:9090`, ou un port non exposé par le répartiteur), avec `/metrics`, `/healthz`,
`/readyz` et `pprof` (niveau 11).

## E3 — L'identifiant hostile

Sans contrôle, l'identifiant de 10 000 caractères part tel quel dans chaque ligne de journal et dans la
réponse ; celui qui contient `\n` **injecte une fausse ligne** (`requête id=faux route="/admin"
status=200`), qui tromperait une enquête. `validID` n'accepte que 1 à 64 lettres, chiffres ou `-`, et
remplace tout le reste par un identifiant neuf ; le test vérifie qu'il n'y a qu'une ligne par requête.

## E4 — La cardinalité cachée

```
séries : message d'erreur=1000 ; raison=1
```

Chaque message d'erreur différent (il contient l'identifiant de l'adhérent) crée une série. L'étiquette
correcte est une **raison** prise dans une liste fermée (`member_not_found`, `too_many_loans`…), dérivée
des erreurs du domaine par `errors.Is` ; le message complet va dans le journal.

## E5 — Les dépendances

`Outgoing` (ci-dessus) compte chaque appel sortant par dépendance, méthode et statut, avec `status="error"`
quand l'appel n'a pas abouti (serveur arrêté). L'URL ne doit pas être une étiquette : elle contient des
identifiants (`/works/1`, `/works/2`…), donc une série par ressource — exactement le problème de E4.

## Exercice intermédiaire — Observer le service de prêts

`TestBusinessMetricsAndSafeLogs` (dans `corr_test.go`) montre le principe, sur un service réduit :

```
http_requests_total{route="POST /members/{id}/loans",status="201"} 1
http_requests_total{route="POST /members/{id}/loans",status="409"} 3
http_requests_total{route="POST /members/{id}/loans",status="500"} 1
loans_refused_total{reason="overdue"} 1
loans_refused_total{reason="too_many_loans"} 2
loans_total 1
erreur id=req-panne cause="connexion à la base impossible"
```

- les refus métier sont comptés par **raison** (liste fermée, fonction `reason`) ; ils ne sont pas
  journalisés comme des erreurs (ce sont des réponses normales) ;
- l'erreur interne est journalisée avec l'identifiant de requête et sa cause, et le test vérifie
  qu'**aucun secret** (mot de passe, DSN) n'apparaît dans les journaux — la cause a été écrite pour ne pas
  en contenir (N8 L1 : ne jamais envelopper une erreur qui contient le DSN).

Pour le service complet, on ajoute de la même façon les compteurs du client résilient (leçon 9) et ceux
du pool (`db.Stats()` lu périodiquement, N8 L5). Trois alertes raisonnables :

1. **Taux d'erreurs 5xx > 1 % pendant 5 min** → page l'astreinte ; consulter les journaux filtrés par
   route et statut, puis les traces.
2. **Durée p99 de `POST /members/{id}/loans` > 300 ms pendant 10 min** (objectif de service) → chercher la
   dépendance lente (métriques sortantes, `WaitDuration` du pool).
3. **Disjoncteur du catalogue ouvert depuis plus de 5 min** → le mode dégradé tient, mais prévenir l'équipe
   du catalogue ; ce n'est pas une urgence pour les emprunts.

## Défi — Une trace minimale

```go
// Fichier : trace/trace.go
// Défi : une trace minimale au format W3C traceparent, sans bibliothèque externe.
package trace

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"net/http"
	"slices"
	"strings"
	"sync"
	"time"
)

type Span struct {
	TraceID, ID, Parent, Name string
	Start                     time.Time
	Duration                  time.Duration
	sampled                   bool
}

type Collector struct {
	mu    sync.Mutex
	spans []Span
}

func (c *Collector) add(s Span) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.spans = append(c.spans, s)
}

type ctxKey struct{}

func hexID(n int) string {
	b := make([]byte, n)
	rand.Read(b)
	return hex.EncodeToString(b)
}

// parse lit « 00-<trace 32 hex>-<parent 16 hex>-<drapeaux 2 hex> ».
func parse(h string) (traceID, parent string, sampled, ok bool) {
	p := strings.Split(h, "-")
	if len(p) != 4 || p[0] != "00" || len(p[1]) != 32 || len(p[2]) != 16 || len(p[3]) != 2 {
		return "", "", false, false
	}
	return p[1], p[2], p[3] == "01", true
}

// Start ouvre un span enfant de celui du contexte (ou une nouvelle trace).
func Start(ctx context.Context, name string) (context.Context, *Span) {
	s := &Span{ID: hexID(8), Name: name, Start: time.Now()}
	if parent, ok := ctx.Value(ctxKey{}).(*Span); ok {
		s.TraceID, s.Parent, s.sampled = parent.TraceID, parent.ID, parent.sampled
	}
	return context.WithValue(ctx, ctxKey{}, s), s
}

func (c *Collector) End(s *Span) {
	s.Duration = time.Since(s.Start)
	if s.sampled {
		c.add(*s)
	}
}

// Middleware : reprend la trace de l'appelant, ou en crée une — et DÉCIDE ici de l'échantillonnage.
func (c *Collector) Middleware(name string, sampleOneIn int, next http.Handler) http.Handler {
	var n int64
	var mu sync.Mutex
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		s := &Span{ID: hexID(8), Name: name + " " + r.Method + " " + r.URL.Path, Start: time.Now()}
		if tid, parent, sampled, ok := parse(r.Header.Get("traceparent")); ok {
			s.TraceID, s.Parent, s.sampled = tid, parent, sampled // décision héritée
		} else {
			mu.Lock()
			n++
			s.sampled = n%int64(sampleOneIn) == 1 || sampleOneIn == 1
			mu.Unlock()
			s.TraceID = hexID(16)
		}
		next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), ctxKey{}, s)))
		c.End(s)
	})
}

// Transport : un span enfant par appel sortant, et l'en-tête traceparent qui le désigne.
type Transport struct {
	Base http.RoundTripper
	C    *Collector
}

func (t Transport) RoundTrip(r *http.Request) (*http.Response, error) {
	ctx, s := Start(r.Context(), "appel "+r.Method+" "+r.URL.Host)
	r = r.Clone(ctx)
	flags := "00"
	if s.sampled {
		flags = "01"
	}
	r.Header.Set("traceparent", fmt.Sprintf("00-%s-%s-%s", s.TraceID, s.ID, flags))
	resp, err := t.Base.RoundTrip(r)
	t.C.End(s)
	return resp, err
}

// Tree affiche l'arbre d'une trace, avec les durées arrondies à 10 ms.
func (c *Collector) Tree(traceID string) string {
	c.mu.Lock()
	defer c.mu.Unlock()
	children := map[string][]Span{}
	var roots []Span
	ids := map[string]bool{}
	for _, s := range c.spans {
		if s.TraceID == traceID {
			ids[s.ID] = true
		}
	}
	for _, s := range c.spans {
		if s.TraceID != traceID {
			continue
		}
		if ids[s.Parent] {
			children[s.Parent] = append(children[s.Parent], s)
		} else {
			roots = append(roots, s)
		}
	}
	var b strings.Builder
	var walk func(s Span, depth int)
	walk = func(s Span, depth int) {
		fmt.Fprintf(&b, "%s%s (%v)\n", strings.Repeat("  ", depth), s.Name, s.Duration.Round(10*time.Millisecond))
		kids := children[s.ID]
		slices.SortFunc(kids, func(a, b Span) int { return a.Start.Compare(b.Start) })
		for _, k := range kids {
			walk(k, depth+1)
		}
	}
	for _, r := range roots {
		walk(r, 0)
	}
	return b.String()
}

func (c *Collector) TraceIDs() []string {
	c.mu.Lock()
	defer c.mu.Unlock()
	seen := map[string]bool{}
	for _, s := range c.spans {
		seen[s.TraceID] = true
	}
	var out []string
	for id := range seen {
		out = append(out, id)
	}
	return out
}
```

```go
// Fichier : trace/trace_test.go
package trace

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

func TestTraceAcrossServices(t *testing.T) {
	c := &Collector{}
	client := &http.Client{Transport: Transport{Base: http.DefaultTransport, C: c}}
	db := func(r *http.Request) { // « base simulée » : un span local
		_, s := Start(r.Context(), "requête SQL")
		time.Sleep(30 * time.Millisecond)
		c.End(s)
	}
	catalog := httptest.NewServer(c.Middleware("catalogue", 1, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		db(r)
	})))
	defer catalog.Close()
	loans := httptest.NewServer(c.Middleware("prêt", 1, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(10 * time.Millisecond)
		req, _ := http.NewRequestWithContext(r.Context(), "GET", catalog.URL+"/works/1", nil)
		resp, err := client.Do(req)
		if err == nil {
			resp.Body.Close()
		}
	})))
	defer loans.Close()

	resp, _ := http.Get(loans.URL + "/loans/42")
	resp.Body.Close()
	ids := c.TraceIDs()
	if len(ids) != 1 {
		t.Fatalf("%d traces, attendu 1 (une requête)", len(ids))
	}
	tree := c.Tree(ids[0])
	t.Log("\n" + tree)
	lines := strings.Split(strings.TrimSpace(tree), "\n")
	if len(lines) != 4 || !strings.HasPrefix(lines[0], "prêt") || !strings.HasPrefix(lines[3], "      requête SQL") {
		t.Fatalf("arbre inattendu :\n%s", tree)
	}
}

func TestSamplingDecidedOnce(t *testing.T) {
	c := &Collector{}
	client := &http.Client{Transport: Transport{Base: http.DefaultTransport, C: c}}
	down := httptest.NewServer(c.Middleware("aval", 1, http.HandlerFunc(func(http.ResponseWriter, *http.Request) {})))
	defer down.Close()
	up := httptest.NewServer(c.Middleware("amont", 10, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		req, _ := http.NewRequestWithContext(r.Context(), "GET", down.URL, nil)
		if resp, err := client.Do(req); err == nil {
			resp.Body.Close()
		}
	})))
	defer up.Close()
	for range 100 {
		resp, _ := http.Get(up.URL)
		resp.Body.Close()
	}
	// 10 traces conservées sur 100, et chacune COMPLÈTE (amont, appel, aval) : l'aval, qui
	// échantillonnerait tout seul à 100 %, respecte la décision prise à l'entrée.
	ids := c.TraceIDs()
	for _, id := range ids {
		if n := strings.Count(c.Tree(id), "\n"); n != 3 {
			t.Fatalf("trace incomplète (%d spans)", n)
		}
	}
	if len(ids) != 10 {
		t.Fatalf("%d traces conservées, attendu 10", len(ids))
	}
}
```

Mesuré, une requête qui traverse « prêt » → « catalogue » → « base simulée » :

```
prêt GET /loans/42 (40ms)
  appel GET 127.0.0.1:35167 (30ms)
    catalogue GET /works/1 (30ms)
      requête SQL (30ms)
```

L'arbre montre d'un coup d'œil que 30 ms sur 40 sont passées dans la requête SQL du catalogue — ce qu'aucune
métrique agrégée du service « prêt » ne dit.

**Échantillonnage.** Le test `TestSamplingDecidedOnce` conserve 10 traces sur 100, et chacune est
**complète** (amont, appel, aval). La décision doit être prise **à l'entrée** et propagée (dernier champ de
`traceparent`, `01` ou `00`) : si chaque service décidait seul, on conserverait des morceaux de traces
différentes — l'amont de l'une, l'aval d'une autre —, inutilisables.

**Ce qui reste pour un usage réel** (ce qu'apporte OpenTelemetry, niveau 12) : l'export vers un
collecteur (protocole OTLP), les attributs standardisés (méthode, statut, route), les événements et
erreurs attachés aux spans, l'instrumentation toute faite des clients et serveurs (HTTP, gRPC, base),
l'échantillonnage configurable (y compris *a posteriori*, pour garder toutes les traces en erreur), et le
lien entre traces, métriques et journaux.

## Quiz

1. Métriques : est-ce que ça va ? Journaux : que s'est-il passé pour cette requête ? Traces : où est passé
   le temps ?
2. Pour instrumenter une fois, sans toucher au métier : requêtes entrantes, appels sortants, messages,
   tâches de fond.
3. RED : taux, erreurs, durées d'un service ; USE : utilisation, saturation, erreurs d'une ressource.
4. La moyenne cache la distribution ; l'histogramme permet les percentiles.
5. Un identifiant attribué à l'entrée d'une requête, mis dans le contexte, écrit dans chaque journal et
   envoyé dans les en-têtes des appels sortants.
6. Le nombre de séries distinctes ; éviter identifiants, chemins bruts, adresses, messages d'erreur.
7. Le motif de la route choisie par le `ServeMux` ; rempli pendant `ServeHTTP` (vide pour un 404 du routeur).
8. Les symptômes sont ce que subissent les utilisateurs ; une cause (processeur élevé) n'est souvent pas un
   problème.
9. Un objectif chiffré (part de requêtes réussies sous un seuil) ; l'écart toléré est le budget d'erreurs.
10. Parce qu'elle dépend d'interfaces aux frontières et d'un contexte propagé partout : difficile à
    ajouter après coup.
