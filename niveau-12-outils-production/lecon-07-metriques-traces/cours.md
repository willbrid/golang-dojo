# Leçon 7 — Métriques et traces

> La leçon 10 du niveau 10 a décidé **quoi** observer : les signaux RED, des étiquettes à faible cardinalité,
> un identifiant propagé d'un service à l'autre. Elle le faisait avec un registre écrit à la main. Cette
> leçon passe aux outils standards de l'industrie : **Prometheus** pour les métriques et **OpenTelemetry**
> pour les traces. Elle relie les trois signaux, métriques, traces et journaux, par un même identifiant :
> une alerte sur une métrique mène à une trace, et la trace aux lignes de journal de chaque service
> traversé.

## Objectifs

- Exposer des métriques Prometheus : compteurs, jauges, histogrammes, étiquettes, registre.
- Choisir les bornes d'un histogramme, et savoir en tirer un percentile.
- Tracer une requête à travers deux services avec OpenTelemetry : fournisseur, spans, propagation,
  échantillonnage.
- Ajouter `trace_id` et `span_id` à chaque ligne de journal.
- Mesurer ce que coûte l'instrumentation.

## Explication

### Prometheus : le modèle

Un serveur Prometheus **interroge** (*scrape*) périodiquement, par exemple toutes les 15 s, l'URL
`/metrics` de chaque instance. Il reçoit un texte de la forme :

```
# TYPE http_requests_total counter
http_requests_total{route="GET /books/{id}",service="catalogue",status="200"} 2
```

Chaque combinaison de nom et d'étiquettes est une **série temporelle**. Prometheus les conserve et permet de
les interroger avec son langage, PromQL.

| Type | Ce qu'il représente | Opérations | Exemple |
|---|---|---|---|
| compteur (*counter*) | un total qui ne fait que croître (remis à zéro au redémarrage) | `Inc`, `Add` | requêtes, erreurs, octets envoyés |
| jauge (*gauge*) | une valeur qui monte et descend | `Set`, `Inc`, `Dec` | requêtes en cours, taille d'une file |
| histogramme | une distribution : un compteur par intervalle (*bucket*), plus la somme et le nombre | `Observe` | durées, tailles |

On n'expose jamais un **taux** : on expose un compteur, et Prometheus calcule le taux
(`rate(http_requests_total[5m])`) à partir de deux relevés. C'est ce qui rend le comptage robuste. Un relevé
manqué ou un redémarrage ne fausse pas le calcul.

Les conventions de nommage :

- **un suffixe d'unité** au pluriel, en unité de base : `_seconds`, `_bytes`, jamais `_ms` ;
- **`_total`** pour un compteur ;
- **des étiquettes en minuscules**.

Les étiquettes suivent la règle de **cardinalité** de N10 L10 : la route (le **motif**), le statut, le
service, et jamais un identifiant, un chemin brut ou une adresse e-mail.

### Les histogrammes et les percentiles

Un histogramme Prometheus compte les observations **inférieures ou égales** à chaque borne (`le`, *less or
equal*). PromQL en déduit un percentile **approché** :

```
histogram_quantile(0.99, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))
```

La précision dépend des **bornes**. Si toutes les requêtes durent entre 100 et 250 ms et que les bornes
sont 0,1 et 0,25, le p99 calculé est une interpolation dans cet intervalle. On choisit donc les bornes
**autour de l'objectif** : si l'objectif est un p99 sous 250 ms, il faut plusieurs bornes entre 50 ms et
500 ms. Les bornes par défaut (de 5 ms à 10 s) conviennent à un service web ordinaire.

Les histogrammes s'**agrègent** : on additionne les compteurs de toutes les instances, puis on calcule le
percentile global. Les percentiles ne se moyennent pas, eux (N11 L7). C'est la raison de préférer
l'histogramme au type *summary*, qui calcule ses percentiles dans chaque instance.

Chaque borne est une série de plus : 10 bornes × 20 routes, cela fait 200 séries, multipliées par le nombre
d'instances. Il faut rester sobre.

### Prometheus en Go

La bibliothèque officielle est `github.com/prometheus/client_golang` (v1.25.0 ici).

- **Un registre à soi** (`prometheus.NewRegistry()`), plutôt que le registre global : il est testable, et
  aucune bibliothèque n'y enregistre de métriques à notre insu.
- **Les métriques avec étiquettes** se déclarent en `CounterVec` et `HistogramVec`. `WithLabelValues(route,
  status)` renvoie la série.
- **`collectors.NewGoCollector()`** expose le *runtime* : goroutines, tas, cycles du GC, à partir de
  `runtime/metrics` (N11 L3). **`collectors.NewProcessCollector`** expose processeur, mémoire et descripteurs
  du processus.
- **`promhttp.HandlerFor(reg, …)`** sert `/metrics`, sur l'adresse d'**administration** (N10 L10), comme
  `pprof` (N11 L4).
- **`prometheus/testutil`** permet de lire une valeur dans un test (`ToFloat64`) et de compter les séries
  (`CollectAndCount`).

### Les traces : le modèle

Une **trace** est l'histoire d'une requête à travers le système. Elle est faite de **spans**, chacun
représentant une opération : la requête reçue par le catalogue, l'appel au service de stock, la requête
reçue par le stock, une étape interne.

Un span a :
- un nom, un début et une fin ;
- des attributs (`book.id=42`) ;
- un statut ;
- l'identifiant de la trace et celui de son span **parent**.

On reconstruit ainsi l'arbre des appels, et l'on voit où le temps a été passé.

Le **contexte de trace** (identifiant de trace, identifiant du span courant, décision d'échantillonnage)
voyage :
- dans un processus, par le `context.Context` (N6 L8) ;
- d'un service à l'autre, par l'en-tête W3C `traceparent`, déjà écrit à la main en N10 L10.

### OpenTelemetry en Go

OpenTelemetry est le standard ouvert de l'instrumentation : une API, un SDK et un protocole d'export (OTLP),
reçus par tous les outils du marché (Jaeger, Tempo, les services des fournisseurs de nuage). Les modules :

| Module | Rôle |
|---|---|
| `go.opentelemetry.io/otel` (v1.47.0) | l'API : `Tracer`, `Span`, propagateurs |
| `go.opentelemetry.io/otel/sdk` | l'implémentation : `TracerProvider`, échantillonneurs, export par lots |
| `…/exporters/stdout/stdouttrace` | un exportateur qui écrit les spans en JSON (démonstration, tests) ; en production : `…/exporters/otlp/otlptrace/otlptracegrpc` vers un collecteur |
| `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp` (v0.72.0) | l'instrumentation HTTP prête à l'emploi : un span par requête entrante (`NewHandler`) et sortante (`NewTransport`), avec propagation de `traceparent` |

Le **fournisseur** (`TracerProvider`) se configure une fois dans `main` :
- l'exportateur, par lots (`WithBatcher`) : les spans sont envoyés en arrière-plan, pas pendant la requête ;
- la **ressource** : le nom du service ;
- l'**échantillonneur**.

On l'arrête avec `Shutdown` à la fin du programme, pour vider les spans en attente.

**L'échantillonnage.** Tracer 100 % des requêtes coûte cher : temps de processeur, réseau, stockage. On garde
une fraction des **nouvelles** traces, avec `TraceIDRatioBased(0.05)`, soit 5 %. Une requête qui arrive avec
une trace déjà commencée suit la décision de l'amont, avec `ParentBased(…)`. Sans cela, une trace serait
gardée dans un service et pas dans l'autre, et elle serait incomplète.

Un premier essai de l'exemple nommait les spans d'après le **chemin** (`GET /books/42`) : un nom par livre,
la même faute de cardinalité que pour les métriques. `otelhttp` fait mieux par défaut. Il nomme le span
d'après le **motif** de route (`GET /books/{id}`), qu'il lit dans `r.Pattern` après le passage dans le
`ServeMux`.

### Corréler les trois signaux

Le gestionnaire de journaux de la leçon 6 sait ajouter des attributs tirés du contexte. `TraceHandler` y
ajoute `trace_id` et `span_id`, lus par `trace.SpanContextFromContext(ctx)`. Le chemin d'un diagnostic
devient :

1. une **métrique** déclenche l'alerte : le p99 de `GET /books/{id}` dépasse 250 ms ;
2. on ouvre une **trace** lente de cette route, dans l'outil de traces : le temps est passé dans l'appel au
   stock ;
3. on filtre les **journaux** sur son `trace_id` : on y lit l'erreur du stock, dans le service de stock.

L'identifiant de requête de N10 L10 devient inutile : le `trace_id` joue ce rôle, et il est propagé par
OpenTelemetry.

### Le coût

Mesuré dans l'exemple (un handler qui ne fait rien, pour isoler le coût de l'instrumentation) :

| Instrumentation | Temps par requête | Allocations |
|---|---|---|
| aucune | 1,15 µs | 11 |
| métriques (compteur + histogramme + jauge) | 1,56 µs (+0,4) | 13 |
| `otelhttp`, 100 % des traces | 9,47 µs (+8,3) | 75 |
| `otelhttp`, 0 % des traces | 5,00 µs (+3,9) | 42 |
| métriques + traces à 100 % | 10,9 µs | 80 |

- **Les métriques sont presque gratuites** : on les met partout.
- **Une trace coûte environ 8 µs par requête** dans le service, sans compter l'export, qui se fait en
  arrière-plan dans un autre fil et n'est donc pas mesuré ici. C'est négligeable pour un service qui fait
  des requêtes en base de quelques millisecondes, et réel pour un service qui répond en 50 µs.
- **Même à 0 %, `otelhttp` coûte 3,9 µs** : il lit l'en-tête, crée un span non enregistré, enveloppe le
  `ResponseWriter`. L'échantillonnage réduit le coût de l'export, pas tout celui de l'instrumentation.

## Exemple

Deux services, `catalogue` et `stock`, chacun avec ses métriques, son fournisseur de traces et son logger.
Une requête `GET /books/42` sur le catalogue appelle `GET /stock/42` sur le stock. Les deux services
tournent dans le même processus (`httptest`), pour que tout tienne dans un programme.

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27

require (
	github.com/prometheus/client_golang v1.25.0
	go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.72.0
	go.opentelemetry.io/otel v1.47.0
	go.opentelemetry.io/otel/exporters/stdout/stdouttrace v1.47.0
	go.opentelemetry.io/otel/sdk v1.47.0
	go.opentelemetry.io/otel/trace v1.47.0
)
```

(Les dépendances indirectes, ajoutées par `go mod tidy`, ne sont pas recopiées.)

```go
// Fichier : internal/obs/metrics.go
// Package obs : métriques Prometheus, traces OpenTelemetry, et leur corrélation avec les journaux.
package obs

import (
	"net/http"
	"strconv"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/collectors"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// Metrics regroupe les métriques RED du service (N10 L10), dans un registre à lui.
type Metrics struct {
	reg      *prometheus.Registry
	requests *prometheus.CounterVec
	duration *prometheus.HistogramVec
	inFlight prometheus.Gauge
}

func NewMetrics(service string) *Metrics {
	reg := prometheus.NewRegistry() // pas le registre global : testable, et rien d'imprévu dedans
	m := &Metrics{
		reg: reg,
		requests: prometheus.NewCounterVec(prometheus.CounterOpts{
			Name:        "http_requests_total",
			Help:        "Requêtes HTTP terminées, par route et statut.",
			ConstLabels: prometheus.Labels{"service": service},
		}, []string{"route", "status"}),
		duration: prometheus.NewHistogramVec(prometheus.HistogramOpts{
			Name:        "http_request_duration_seconds",
			Help:        "Durée des requêtes HTTP.",
			ConstLabels: prometheus.Labels{"service": service},
			// Des bornes choisies autour de l'objectif (p99 < 250 ms) : la précision est là où elle sert.
			Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5},
		}, []string{"route"}),
		inFlight: prometheus.NewGauge(prometheus.GaugeOpts{
			Name:        "http_requests_in_flight",
			Help:        "Requêtes en cours.",
			ConstLabels: prometheus.Labels{"service": service},
		}),
	}
	reg.MustRegister(m.requests, m.duration, m.inFlight,
		collectors.NewGoCollector(),                                       // GC, goroutines, mémoire : runtime/metrics
		collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}), // processeur, mémoire, descripteurs
	)
	return m
}

// Handler expose /metrics. À monter sur l'adresse d'ADMINISTRATION, jamais sur l'adresse publique.
func (m *Metrics) Handler() http.Handler {
	return promhttp.HandlerFor(m.reg, promhttp.HandlerOpts{})
}

type statusWriter struct {
	http.ResponseWriter
	status int
}

func (s *statusWriter) WriteHeader(code int) { s.status = code; s.ResponseWriter.WriteHeader(code) }

// Middleware mesure chaque requête. L'étiquette route est le MOTIF (r.Pattern), jamais le chemin.
func (m *Metrics) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		m.inFlight.Inc()
		defer m.inFlight.Dec()
		sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}
		start := time.Now()
		next.ServeHTTP(sw, r)
		route := r.Pattern
		if route == "" {
			route = "aucune"
		}
		m.requests.WithLabelValues(route, strconv.Itoa(sw.status)).Inc()
		m.duration.WithLabelValues(route).Observe(time.Since(start).Seconds())
	})
}
```

```go
// Fichier : internal/obs/tracing.go
package obs

import (
	"context"
	"io"
	"log/slog"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.43.0"
	"go.opentelemetry.io/otel/trace"
)

// NewTracerProvider crée le fournisseur de traces d'un service. Ici, les spans sont écrits en JSON
// sur w (exportateur stdout) ; en production, on utilise l'exportateur OTLP vers un collecteur.
func NewTracerProvider(service string, w io.Writer, ratio float64) (*sdktrace.TracerProvider, error) {
	exp, err := stdouttrace.New(stdouttrace.WithWriter(w))
	if err != nil {
		return nil, err
	}
	return sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exp),
		sdktrace.WithResource(resource.NewSchemaless(semconv.ServiceName(service))),
		// On garde une fraction des traces NOUVELLES ; une trace commencée en amont suit la décision de l'amont.
		sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(ratio))),
	), nil
}

// Propagation W3C (en-tête traceparent, N10 L10), réglée une fois pour tout le processus.
func init() {
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{}, propagation.Baggage{}))
}

// TraceHandler ajoute trace_id et span_id à chaque ligne de journal écrite avec un contexte tracé.
type TraceHandler struct{ slog.Handler }

func (h TraceHandler) Handle(ctx context.Context, r slog.Record) error {
	if sc := trace.SpanContextFromContext(ctx); sc.IsValid() {
		r.AddAttrs(slog.String("trace_id", sc.TraceID().String()), slog.String("span_id", sc.SpanID().String()))
	}
	return h.Handler.Handle(ctx, r)
}

func (h TraceHandler) WithAttrs(a []slog.Attr) slog.Handler {
	return TraceHandler{h.Handler.WithAttrs(a)}
}
func (h TraceHandler) WithGroup(n string) slog.Handler { return TraceHandler{h.Handler.WithGroup(n)} }
```

```go
// Fichier : internal/obs/obs_test.go
package obs

import (
	"bytes"
	"context"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"

	"github.com/prometheus/client_golang/prometheus/testutil"
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func mux() *http.ServeMux {
	m := http.NewServeMux()
	m.HandleFunc("GET /books/{id}", func(w http.ResponseWriter, r *http.Request) { w.Write([]byte("ok")) })
	return m
}

func TestMetricsMiddleware(t *testing.T) {
	m := NewMetrics("test")
	h := m.Middleware(mux())
	for _, p := range []string{"/books/1", "/books/2", "/books/3", "/nope"} {
		h.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", p, nil))
	}
	if got := testutil.ToFloat64(m.requests.WithLabelValues("GET /books/{id}", "200")); got != 3 {
		t.Errorf("route connue : %v", got)
	}
	if got := testutil.ToFloat64(m.requests.WithLabelValues("aucune", "404")); got != 1 {
		t.Errorf("404 : %v", got)
	}
	// Trois livres différents, UNE seule série : la cardinalité ne dépend pas des identifiants.
	if n := testutil.CollectAndCount(m.requests); n != 2 {
		t.Errorf("%d séries, attendu 2", n)
	}
}

func TestTraceIDInLogs(t *testing.T) {
	tp, err := NewTracerProvider("test", io.Discard, 1)
	if err != nil {
		t.Fatal(err)
	}
	defer tp.Shutdown(context.Background())
	var buf bytes.Buffer
	log := slog.New(TraceHandler{Handler: slog.NewTextHandler(&buf, nil)})
	ctx, span := tp.Tracer("t").Start(context.Background(), "op")
	log.InfoContext(ctx, "dans le span")
	span.End()
	log.Info("hors span")
	lines := strings.Split(strings.TrimSpace(buf.String()), "\n")
	if !strings.Contains(lines[0], "trace_id="+span.SpanContext().TraceID().String()) {
		t.Errorf("trace_id absent : %s", lines[0])
	}
	if strings.Contains(lines[1], "trace_id") {
		t.Errorf("trace_id inattendu : %s", lines[1])
	}
}

// Le coût de l'instrumentation, par requête.
func BenchmarkInstrumentation(b *testing.B) {
	req := httptest.NewRequest("GET", "/books/42", nil)
	m := NewMetrics("bench")
	tp1, _ := NewTracerProvider("bench", io.Discard, 1)
	tp0, _ := NewTracerProvider("bench", io.Discard, 0)
	defer tp1.Shutdown(context.Background())
	defer tp0.Shutdown(context.Background())
	for _, tc := range []struct {
		name string
		h    http.Handler
	}{
		{"nu", mux()},
		{"métriques", m.Middleware(mux())},
		{"traces-100%", otelhttp.NewHandler(mux(), "s", otelhttp.WithTracerProvider(tp1))},
		{"traces-0%", otelhttp.NewHandler(mux(), "s", otelhttp.WithTracerProvider(tp0))},
		{"métriques+traces-100%", otelhttp.NewHandler(m.Middleware(mux()), "s", otelhttp.WithTracerProvider(tp1))},
	} {
		b.Run(tc.name, func(b *testing.B) {
			for b.Loop() {
				tc.h.ServeHTTP(httptest.NewRecorder(), req)
			}
		})
	}
}
```

```go
// Fichier : cmd/demo/main.go
// Commande demo : deux services instrumentés ; une requête traverse les deux.
package main

import (
	"bufio"
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel/attribute"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/trace"

	"example.com/svc/internal/obs"
)

type service struct {
	name    string
	spans   bytes.Buffer // ce que l'exportateur écrit
	metrics *obs.Metrics
	tp      *sdktrace.TracerProvider
	tracer  trace.Tracer
	log     *slog.Logger
	srv     *httptest.Server
}

func newService(name string, logs io.Writer, routes func(s *service, mux *http.ServeMux)) *service {
	s := &service{name: name, metrics: obs.NewMetrics(name)}
	tp, err := obs.NewTracerProvider(name, &s.spans, 1.0) // 100 % des traces, pour la démonstration
	if err != nil {
		log.Fatal(err)
	}
	s.tp, s.tracer = tp, tp.Tracer("example.com/svc")
	s.log = slog.New(obs.TraceHandler{Handler: slog.NewTextHandler(logs, &slog.HandlerOptions{
		ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
			if a.Key == slog.TimeKey {
				return slog.Attr{}
			}
			return a
		}})}).With("service", name)
	mux := http.NewServeMux()
	routes(s, mux)
	// otelhttp : un span par requête entrante, contexte de trace repris de l'en-tête traceparent.
	h := otelhttp.NewHandler(s.metrics.Middleware(mux), "http.server",
		otelhttp.WithTracerProvider(tp))
	s.srv = httptest.NewServer(h)
	return s
}

func main() {
	var logs bytes.Buffer
	stock := newService("stock", &logs, func(s *service, mux *http.ServeMux) {
		mux.HandleFunc("GET /stock/{id}", func(w http.ResponseWriter, r *http.Request) {
			s.log.InfoContext(r.Context(), "stock lu", "book_id", r.PathValue("id"))
			fmt.Fprintln(w, 3)
		})
	})
	catalog := newService("catalogue", &logs, func(s *service, mux *http.ServeMux) {
		// Le client sortant crée un span client et propage le contexte de trace (en-tête traceparent).
		client := &http.Client{Transport: otelhttp.NewTransport(http.DefaultTransport, otelhttp.WithTracerProvider(s.tp))}
		mux.HandleFunc("GET /books/{id}", func(w http.ResponseWriter, r *http.Request) {
			ctx, span := s.tracer.Start(r.Context(), "charger le livre") // un span à nous, dans la requête
			span.SetAttributes(attribute.String("book.id", r.PathValue("id")))
			s.log.InfoContext(ctx, "livre chargé", "book_id", r.PathValue("id"))
			span.End()
			req, _ := http.NewRequestWithContext(r.Context(), "GET", stock.srv.URL+"/stock/"+r.PathValue("id"), nil)
			resp, err := client.Do(req)
			if err != nil {
				http.Error(w, "stock indisponible", http.StatusBadGateway)
				return
			}
			resp.Body.Close()
			fmt.Fprintln(w, "ok")
		})
	})
	for _, id := range []string{"42", "7"} {
		resp, err := http.Get(catalog.srv.URL + "/books/" + id)
		if err != nil {
			log.Fatal(err)
		}
		resp.Body.Close()
	}
	catalog.srv.Close()
	stock.srv.Close()
	for _, s := range []*service{catalog, stock} {
		s.tp.Shutdown(context.Background()) // vide le lot de spans en attente vers l'exportateur
	}

	fmt.Println("== journaux")
	fmt.Print(logs.String())
	fmt.Println("== spans")
	for _, s := range []*service{catalog, stock} {
		printSpans(s.name, &s.spans)
	}
	fmt.Println("== /metrics du catalogue (extrait)")
	rec := httptest.NewRecorder()
	catalog.metrics.Handler().ServeHTTP(rec, httptest.NewRequest("GET", "/metrics", nil))
	sc := bufio.NewScanner(rec.Body)
	for sc.Scan() {
		l := sc.Text()
		if strings.HasPrefix(l, "http_requests_total") || strings.HasPrefix(l, "# TYPE http_") ||
			strings.Contains(l, `le="0.01"`) || strings.Contains(l, `le="+Inf"`) ||
			strings.HasPrefix(l, "http_request_duration_seconds_count") || strings.HasPrefix(l, "go_goroutines") {
			fmt.Println(l)
		}
	}
}

func printSpans(service string, r io.Reader) {
	dec := json.NewDecoder(r)
	for {
		var s struct {
			Name        string
			SpanContext struct{ TraceID, SpanID string }
			Parent      struct{ SpanID string }
			SpanKind    int
		}
		if err := dec.Decode(&s); err != nil {
			return
		}
		fmt.Printf("%-9s %-24q trace=%s… span=%s parent=%s\n", service, s.Name,
			s.SpanContext.TraceID[:8], s.SpanContext.SpanID[:8], short(s.Parent.SpanID))
	}
}

func short(id string) string {
	if id == "" || strings.Trim(id, "0") == "" {
		return "(racine)"
	}
	return id[:8]
}
```

```
$ go run ./cmd/demo
== journaux
level=INFO msg="livre chargé" service=catalogue book_id=42 trace_id=42fc239b3b07139bfef1f0dfdb393ad0 span_id=a5dd26b6e96f2c6f
level=INFO msg="stock lu" service=stock book_id=42 trace_id=42fc239b3b07139bfef1f0dfdb393ad0 span_id=d615a6e2813644f1
level=INFO msg="livre chargé" service=catalogue book_id=7 trace_id=0c6cfbbff97d0ebe95cb1387df09e7a6 span_id=983338f43587e69b
level=INFO msg="stock lu" service=stock book_id=7 trace_id=0c6cfbbff97d0ebe95cb1387df09e7a6 span_id=322d72e0e56ca2cb
== spans
catalogue "charger le livre"       trace=42fc239b… span=a5dd26b6 parent=31ed8892
catalogue "HTTP GET"               trace=42fc239b… span=d94915f6 parent=31ed8892
catalogue "GET /books/{id}"        trace=42fc239b… span=31ed8892 parent=(racine)
catalogue "charger le livre"       trace=0c6cfbbf… span=983338f4 parent=baed9187
catalogue "HTTP GET"               trace=0c6cfbbf… span=f664ecb7 parent=baed9187
catalogue "GET /books/{id}"        trace=0c6cfbbf… span=baed9187 parent=(racine)
stock     "GET /stock/{id}"        trace=42fc239b… span=d615a6e2 parent=d94915f6
stock     "GET /stock/{id}"        trace=0c6cfbbf… span=322d72e0 parent=f664ecb7
== /metrics du catalogue (extrait)
go_goroutines 4
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{route="GET /books/{id}",service="catalogue",le="0.01"} 2
http_request_duration_seconds_bucket{route="GET /books/{id}",service="catalogue",le="+Inf"} 2
http_request_duration_seconds_count{route="GET /books/{id}",service="catalogue"} 2
# TYPE http_requests_in_flight gauge
# TYPE http_requests_total counter
http_requests_total{route="GET /books/{id}",service="catalogue",status="200"} 2
```

```
$ go test -v ./internal/obs/
--- PASS: TestMetricsMiddleware (0.00s)
--- PASS: TestTraceIDInLogs (0.00s)
ok  	example.com/svc/internal/obs	0.005s
```

**Lecture de la sortie :**

- **Les journaux** des deux services portent le **même** `trace_id` pour une même requête
  (`dad6c5a9…` pour le livre 42), avec un `span_id` différent. On les retrouve ensemble par un seul filtre.
- **Les spans** forment un arbre :
  - `GET /books/{id}` (catalogue, racine) a pour enfants `charger le livre` (notre span) et `HTTP GET`
    (l'appel sortant, créé par `NewTransport`) ;
  - `GET /stock/{id}` (stock) a pour **parent** ce span `HTTP GET`. Le parent vient d'un autre service : il
    est arrivé par l'en-tête `traceparent`.
- **Les métriques** : deux livres différents, mais **une** série `route="GET /books/{id}"`. Les deux requêtes
  ont duré moins de 10 ms (`le="0.01"` vaut 2).

Les identifiants de trace sont aléatoires : ils changent à chaque exécution.

## Explication du code

| Élément | Rôle |
|---|---|
| `prometheus.NewRegistry()` | Un registre par service ; `MustRegister` panique au démarrage si deux métriques ont le même nom, ce qui est voulu. |
| `ConstLabels: {"service": …}` | Une étiquette fixe, la même pour toutes les séries du service. En vrai, c'est souvent Prometheus qui l'ajoute, d'après la cible interrogée. |
| `Buckets: […]` | Des bornes resserrées autour de l'objectif de latence. |
| `m.inFlight.Inc()` / `defer Dec()` | Une jauge : le nombre de requêtes en cours (la loi de Little de N11 L7 relie cette valeur au débit et à la latence). |
| `route := r.Pattern` | Le motif rempli par le `ServeMux`, lu **après** `next.ServeHTTP`. `aucune` pour un 404 du routeur. |
| `sdktrace.WithBatcher(exp)` | Export par lots, en arrière-plan. `WithSyncer` (export à chaque span) ne sert qu'aux tests. |
| `ParentBased(TraceIDRatioBased(ratio))` | Échantillonnage cohérent d'un service à l'autre. |
| `otel.SetTextMapPropagator(…)` | Le format de propagation : W3C `traceparent` (et `baggage`). À régler **une fois** par processus, d'où le `init`. |
| `otelhttp.NewHandler(…, WithTracerProvider(tp))` | Un span par requête entrante, nommé d'après `r.Pattern`. Le fournisseur est passé explicitement plutôt que par la variable globale d'`otel`, pour que deux services coexistent dans le même processus. |
| `otelhttp.NewTransport(…)` | Un span client par appel sortant, et l'en-tête `traceparent` ajouté à la requête. La requête doit être créée avec le contexte de la requête entrante (`NewRequestWithContext(r.Context(), …)`). |
| `s.tracer.Start(ctx, "charger le livre")` puis `span.End()` | Un span à soi, enfant du span courant. On utilise le **nouveau** contexte (`ctx`) pour journaliser : la ligne porte alors le `span_id` de ce span. |
| `tp.Shutdown(ctx)` | Vide les spans en attente. Sans lui, les derniers spans d'un programme qui s'arrête sont perdus. |
| `TraceHandler` | Ajoute `trace_id` et `span_id` aux journaux, avec `WithAttrs`/`WithGroup` comme en leçon 6. |
| `testutil.CollectAndCount` | Compte les séries : le test vérifie la cardinalité, pas seulement les valeurs. |

## Erreurs fréquentes

1. **Un identifiant, un chemin brut ou un e-mail en étiquette**, ou en nom de span.
2. **Exposer un taux ou une moyenne** au lieu d'un compteur et d'un histogramme.
3. **Des unités dans tous les sens** (`_ms`, `_secs`) au lieu de l'unité de base dans le nom.
4. **Le registre global** et `promhttp.Handler()` sans réfléchir : des métriques venues de bibliothèques, des
   tests qui interfèrent.
5. **`/metrics` sur l'adresse publique.**
6. **Créer une requête sortante sans le contexte entrant** : la trace s'arrête au premier service.
7. **Oublier `tp.Shutdown`** : les derniers spans disparaissent.
8. **Échantillonner sans `ParentBased`** : des traces à trous.
9. **100 % des traces en production** sans avoir mesuré le coût et le volume.
10. **Journaliser avec `log.Info` sans contexte** dans un code tracé : pas de `trace_id`.

## Bonnes pratiques Go

- Métriques RED par un middleware : compteur `_total` par route et statut, histogramme `_seconds` par route,
  jauge des requêtes en cours.
- Un registre par service, les collecteurs Go et processus, `/metrics` sur l'adresse d'administration.
- Des bornes d'histogramme choisies d'après les objectifs.
- OpenTelemetry : fournisseur configuré dans `main`, export par lots vers un collecteur (OTLP),
  `ParentBased(TraceIDRatioBased(…))`, `Shutdown` à l'arrêt.
- `otelhttp` pour les requêtes entrantes et sortantes ; des spans à soi pour les étapes importantes
  seulement.
- `trace_id` et `span_id` dans chaque ligne de journal.
- Tester la cardinalité (`CollectAndCount`) et la présence du `trace_id` dans les journaux.

## Ce que je dois retenir

- Prometheus interroge `/metrics` ; compteurs, jauges et histogrammes ; le taux se calcule, il ne s'expose
  pas.
- Les percentiles se calculent depuis des histogrammes agrégés ; leur précision dépend des bornes.
- Une trace est un arbre de spans, propagé par le contexte et par l'en-tête `traceparent`.
- `ParentBased` garde des traces complètes ; `Shutdown` vide les spans en attente.
- Un même `trace_id` relie l'alerte, la trace et les journaux.
- Métriques : +0,4 µs par requête ; traces : +8 µs, et +4 µs même à 0 %.

---

➡️ [Exercices](exercices.md)
