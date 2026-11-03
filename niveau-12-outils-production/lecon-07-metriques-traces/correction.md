# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les exercices faciles et l'exercice intermédiaire sont des tests du paquet `obs`. Le défi ajoute une commande
`svc` et l'exportateur OTLP ; le module devient :

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27

require (
	github.com/prometheus/client_golang v1.25.0
	go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.72.0
	go.opentelemetry.io/otel v1.47.0
	go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp v1.47.0
	go.opentelemetry.io/otel/exporters/stdout/stdouttrace v1.47.0
	go.opentelemetry.io/otel/sdk v1.47.0
	go.opentelemetry.io/otel/trace v1.47.0
)
```

```go
// Fichier : internal/obs/fix_test.go
package obs

import (
	"context"
	"fmt"
	"math/rand/v2"
	"net/http"
	"net/http/httptest"
	"slices"
	"strconv"
	"sync"
	"testing"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"github.com/prometheus/client_golang/prometheus/testutil"
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/sdk/trace/tracetest"
)

// E1 : une jauge calculée à la lecture.
type cache struct {
	mu sync.Mutex
	m  map[string]string
}

func (c *cache) Len() int { c.mu.Lock(); defer c.mu.Unlock(); return len(c.m) }

func TestE1(t *testing.T) {
	reg := prometheus.NewRegistry()
	viewed := prometheus.NewCounter(prometheus.CounterOpts{Name: "books_viewed_total", Help: "Livres consultés."})
	c := &cache{m: map[string]string{}}
	entries := prometheus.NewGaugeFunc(prometheus.GaugeOpts{Name: "cache_entries", Help: "Entrées du cache."},
		func() float64 { return float64(c.Len()) }) // lue au moment de l'interrogation
	reg.MustRegister(viewed, entries)
	for i := range 5 {
		viewed.Inc()
		c.mu.Lock()
		c.m[strconv.Itoa(i%3)] = "x"
		c.mu.Unlock()
	}
	if testutil.ToFloat64(viewed) != 5 || testutil.ToFloat64(entries) != 3 {
		t.Fatalf("viewed=%v entries=%v", testutil.ToFloat64(viewed), testutil.ToFloat64(entries))
	}
}

// E2 : histogram_quantile, à la main.
func quantile(q float64, bounds []float64, counts []float64) float64 {
	// counts[i] : observations <= bounds[i] (cumulés) ; le dernier est +Inf.
	total := counts[len(counts)-1]
	rank := q * total
	i := 0
	for counts[i] < rank {
		i++
	}
	if i == len(bounds) { // dans le dernier intervalle, +Inf : on renvoie la plus grande borne finie
		return bounds[len(bounds)-1]
	}
	lo, below := 0.0, 0.0
	if i > 0 {
		lo, below = bounds[i-1], counts[i-1]
	}
	return lo + (bounds[i]-lo)*(rank-below)/(counts[i]-below)
}

func TestE2(t *testing.T) {
	rng := rand.New(rand.NewPCG(1, 2))
	d := make([]float64, 100_000)
	for i := range d {
		d[i] = 0.080 + rng.Float64()*0.080
	}
	sorted := slices.Sorted(slices.Values(d))
	truth := sorted[int(0.99*float64(len(d)))-1]
	for _, b := range []struct {
		name   string
		bounds []float64
	}{{"DefBuckets", prometheus.DefBuckets}, {"bornes du cours", []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5}},
		{"bornes resserrées", []float64{.08, .1, .12, .14, .15, .16, .2}}} {
		counts := make([]float64, len(b.bounds)+1)
		for _, x := range d {
			for i, bound := range append(slices.Clone(b.bounds), 1e18) {
				if x <= bound {
					counts[i]++
				}
			}
		}
		est := quantile(0.99, b.bounds, counts)
		fmt.Printf("%-18s p99 estimé %6.1f ms, vrai %6.1f ms, erreur %+5.1f ms\n", b.name, est*1000, truth*1000, (est-truth)*1000)
	}
}

// E3 : l'explosion de cardinalité.
func TestE3(t *testing.T) {
	for _, withUser := range []bool{false, true} {
		reg := prometheus.NewRegistry()
		labels := []string{"route", "status"}
		if withUser {
			labels = append(labels, "user_id")
		}
		c := prometheus.NewCounterVec(prometheus.CounterOpts{Name: "http_requests_total", Help: "x"}, labels)
		reg.MustRegister(c)
		for u := range 10_000 {
			v := []string{"GET /books/{id}", "200"}
			if withUser {
				v = append(v, strconv.Itoa(u))
			}
			c.WithLabelValues(v...).Inc()
		}
		rec := httptest.NewRecorder()
		h := promHandler(reg)
		start := time.Now()
		h.ServeHTTP(rec, httptest.NewRequest("GET", "/metrics", nil))
		fmt.Printf("user_id=%-5v séries %5d, /metrics %8d octets en %v\n", withUser,
			testutil.CollectAndCount(c), rec.Body.Len(), time.Since(start).Round(100*time.Microsecond))
	}
}

// E4 et E5 : deux services, et un exportateur en mémoire pour compter les spans.
func twoServices(t *testing.T, sCat, sStock sdktrace.Sampler, withCtx bool, n int) (cat, stock []tracetest.SpanStub) {
	ec, es := tracetest.NewInMemoryExporter(), tracetest.NewInMemoryExporter()
	tpc := sdktrace.NewTracerProvider(sdktrace.WithSyncer(ec), sdktrace.WithSampler(sCat))
	tps := sdktrace.NewTracerProvider(sdktrace.WithSyncer(es), sdktrace.WithSampler(sStock))
	stockSrv := httptest.NewServer(otelhttp.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {}), "stock", otelhttp.WithTracerProvider(tps)))
	defer stockSrv.Close()
	client := &http.Client{Transport: otelhttp.NewTransport(http.DefaultTransport, otelhttp.WithTracerProvider(tpc))}
	cat0 := otelhttp.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		var req *http.Request
		if withCtx {
			req, _ = http.NewRequestWithContext(r.Context(), "GET", stockSrv.URL, nil)
		} else {
			req, _ = http.NewRequest("GET", stockSrv.URL, nil) // E4 : le contexte est perdu
		}
		resp, err := client.Do(req)
		if err == nil {
			resp.Body.Close()
		}
	}), "catalogue", otelhttp.WithTracerProvider(tpc))
	for range n {
		cat0.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/books/1", nil))
	}
	cat, stock = ec.GetSpans(), es.GetSpans() // AVANT Shutdown : celui de l'exportateur en mémoire efface les spans
	tpc.Shutdown(context.Background())
	tps.Shutdown(context.Background())
	return cat, stock
}

func TestE4(t *testing.T) {
	for _, withCtx := range []bool{true, false} {
		cat, stock := twoServices(t, sdktrace.AlwaysSample(), sdktrace.AlwaysSample(), withCtx, 1)
		root := cat[len(cat)-1] // le span serveur se termine en dernier
		fmt.Printf("avec contexte=%-5v  trace catalogue %s…  trace stock %s…  parent du stock valide=%v\n", withCtx,
			root.SpanContext.TraceID().String()[:8], stock[0].SpanContext.TraceID().String()[:8], stock[0].Parent.IsValid())
	}
}

func TestE5(t *testing.T) {
	for _, parent := range []bool{false, true} {
		sc, ss := sdktrace.Sampler(sdktrace.TraceIDRatioBased(0.5)), sdktrace.Sampler(sdktrace.TraceIDRatioBased(0.1))
		if parent {
			sc, ss = sdktrace.ParentBased(sc), sdktrace.ParentBased(ss)
		}
		cat, stock := twoServices(t, sc, ss, true, 1000)
		traces := map[string]bool{}
		for _, s := range cat {
			traces[s.SpanContext.TraceID().String()] = true
		}
		complete, orphans := 0, 0
		seen := map[string]bool{}
		for _, s := range stock {
			id := s.SpanContext.TraceID().String()
			seen[id] = true
			if traces[id] {
				complete++
			} else {
				orphans++
			}
		}
		fmt.Printf("ParentBased=%-5v  traces catalogue %3d  stock %3d  complètes %3d  spans du stock sans parent gardé %3d\n",
			parent, len(traces), len(seen), complete, orphans)
	}
}

func promHandler(reg *prometheus.Registry) http.Handler {
	return promhttp.HandlerFor(reg, promhttp.HandlerOpts{})
}
```

```
$ go test -v -run 'TestE' ./internal/obs/
--- PASS: TestE1 (0.00s)
DefBuckets         p99 estimé  248.0 ms, vrai  159.2 ms, erreur +88.8 ms
bornes du cours    p99 estimé  248.0 ms, vrai  159.2 ms, erreur +88.8 ms
bornes resserrées  p99 estimé  159.2 ms, vrai  159.2 ms, erreur  +0.0 ms
--- PASS: TestE2 (0.03s)
user_id=false séries     1, /metrics      128 octets en 100µs
user_id=true  séries 10000, /metrics   748954 octets en 62.6ms
--- PASS: TestE3 (0.08s)
avec contexte=true   trace catalogue 96d74d34…  trace stock 96d74d34…  parent du stock valide=true
avec contexte=false  trace catalogue ac86aee3…  trace stock f18cea1e…  parent du stock valide=true
--- PASS: TestE4 (0.00s)
ParentBased=false  traces catalogue 528  stock 120  complètes 120  spans du stock sans parent gardé   0
ParentBased=true   traces catalogue 515  stock 515  complètes 515  spans du stock sans parent gardé   0
--- PASS: TestE5 (0.30s)
```

## E1 — Une métrique métier et une jauge calculée

`books_viewed_total` est un compteur **sans** identifiant de livre. « Quels livres sont les plus consultés ? »
est une question pour la base de données ou les journaux, pas pour une métrique.

`NewGaugeFunc` appelle la fonction **au moment où Prometheus interroge**. Avec un `Set` à chaque modification
du cache, il faudrait penser à mettre la jauge à jour partout où le cache change : à l'ajout, à la
suppression, à l'expiration, au vidage. Le premier oubli fausse la valeur pour toujours. La fonction lit la
vraie valeur, sous le verrou du cache. Elle doit rester **rapide**, car elle est appelée à chaque
interrogation.

## E2 — Les bornes et le percentile

Pour des durées entre 80 et 160 ms, le vrai p99 vaut 159,2 ms.

- **Avec les bornes par défaut** (… 0,1, 0,25 …), le p99 tombe dans l'intervalle [100 ms, 250 ms].
  L'interpolation linéaire suppose les observations **uniformément réparties** dans cet intervalle, et place
  le p99 à 248 ms : **+89 ms**, une erreur de 56 %. Une alerte « p99 > 200 ms » se déclencherait à tort.
- **Les bornes du cours** ont le même trou entre 0,1 et 0,25 : même erreur. Elles sont pensées pour un
  objectif de 250 ms, pas pour ce service.
- **Avec des bornes resserrées** autour des valeurs réelles (0,14, 0,15, 0,16), l'erreur disparaît.

Il faut choisir les bornes **d'après les durées observées et l'objectif**, et les revoir quand le service
change. Le défi montre la même erreur sur un vrai Prometheus.

## E3 — L'explosion de cardinalité

| Étiquettes | Séries | Taille de `/metrics` | Temps de production |
|---|---|---|---|
| `route`, `status` | 1 | 128 octets | 0,1 ms |
| `route`, `status`, `user_id` (10 000 utilisateurs) | 10 000 | 749 Ko | 63 ms |

Ce n'est que pour **une** métrique et **une** route, et la série ne disparaît jamais tant que le processus
tourne. Avec l'histogramme (10 bornes), cela ferait 110 000 séries. Prometheus stocke chaque série
indépendamment, sur des semaines : la mémoire et le disque du serveur explosent, et les requêtes
ralentissent. C'est la faute de N10 L10, mesurée ici sur une vraie bibliothèque.

## E4 — La trace cassée

Sans le contexte, l'appel sortant **démarre une nouvelle trace** (`ac86aee3…` contre `f18cea1e…`). Le span du
stock a bien un parent valide, mais c'est le span client `HTTP GET`, lui-même racine d'une autre trace. La
requête est coupée en **deux** traces sans lien : l'outil de traces montrera un catalogue rapide sans appel
sortant, et un appel au stock venu de nulle part. Les journaux du stock porteront un `trace_id` différent de
ceux du catalogue.

Rien ne le signale, parce que rien n'est **faux** du point de vue du code. Une requête sans contexte est
une requête valide, et `otelhttp` fait de son mieux en démarrant une trace. Seul un test qui vérifie le lien
(comme celui-ci), ou un œil sur l'outil de traces, le voit. C'est l'argument pour toujours écrire
`http.NewRequestWithContext(ctx, …)`. `go vet` ne le vérifie pas. Un analyseur dédié, comme `noctx` dans
`golangci-lint`, signale les requêtes créées sans contexte.

## E5 — L'échantillonnage incohérent

- **Sans `ParentBased`**, le catalogue garde 528 traces (environ 50 %) et le stock 120 (environ 10 %). Les
  120 traces du stock sont **toutes** dans le catalogue. `TraceIDRatioBased` décide d'après la **valeur** de
  l'identifiant de trace (en dessous d'un seuil), et cette décision est la même dans tous les services. À
  10 %, on garde donc un sous-ensemble des traces gardées à 50 %.

  Les traces sont pourtant **incomplètes** : 408 traces du catalogue n'ont pas leur partie stock. Et si le
  stock avait le taux le plus **élevé**, il garderait des spans dont le parent a été jeté.
- **Avec `ParentBased`**, le stock suit la décision transmise par `traceparent`. 515 traces dans chaque
  service, toutes complètes : le taux du stock (10 %) n'est plus utilisé que pour les traces qui
  **commencent** chez lui.

La décision doit être prise **une fois**, à l'entrée du système, et propagée.

---

## Exercice intermédiaire — Le service de prêts, observable

La partie vérifiable sans le service complet est la question 4 : un dépôt instrumenté, et un test avec un
exportateur en mémoire.

```go
// Fichier : internal/obs/loans_test.go
package obs

import (
	"context"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/sdk/trace/tracetest"
	"go.opentelemetry.io/otel/trace"
)

// Un dépôt instrumenté : un span par méthode, l'opération en attribut, jamais les valeurs.
type loanRepo struct{ tracer trace.Tracer }

func (r loanRepo) Create(ctx context.Context, borrowerEmail string, bookID int) error {
	ctx, span := r.tracer.Start(ctx, "loans.Create", trace.WithAttributes(
		attribute.String("db.system", "postgresql"),
		attribute.String("db.operation", "INSERT"),
		attribute.String("db.collection.name", "loans"),
	))
	defer span.End()
	_ = ctx
	if bookID == 13 {
		err := fmt.Errorf("livre %d déjà prêté", bookID)
		span.RecordError(err)
		span.SetStatus(codes.Error, "conflit")
		return err
	}
	return nil
}

func TestLoanTrace(t *testing.T) {
	exp := tracetest.NewInMemoryExporter()
	tp := sdktrace.NewTracerProvider(sdktrace.WithSyncer(exp))
	repo := loanRepo{tracer: tp.Tracer("loans")}
	mux := http.NewServeMux()
	mux.HandleFunc("POST /loans", func(w http.ResponseWriter, r *http.Request) {
		if err := repo.Create(r.Context(), "ada@example.com", 13); err != nil {
			http.Error(w, "conflit", http.StatusConflict)
		}
	})
	h := otelhttp.NewHandler(mux, "loans", otelhttp.WithTracerProvider(tp))
	h.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("POST", "/loans", strings.NewReader(`{}`)))

	spans := exp.GetSpans()
	if len(spans) != 2 {
		t.Fatalf("%d spans", len(spans))
	}
	db, server := spans[0], spans[1]
	if server.Name != "POST /loans" || db.Parent.SpanID() != server.SpanContext.SpanID() {
		t.Errorf("arbre inattendu : %q parent de %q ?", server.Name, db.Name)
	}
	if db.Status.Code != codes.Error {
		t.Errorf("statut du span de dépôt : %v", db.Status)
	}
	for _, s := range spans {
		for _, a := range s.Attributes {
			if strings.Contains(a.Value.Emit(), "@") {
				t.Errorf("donnée personnelle dans %s : %s=%s", s.Name, a.Key, a.Value.Emit())
			}
		}
		fmt.Printf("%-14s parent=%-16s statut=%-5s attributs=%d\n", s.Name, s.Parent.SpanID(), s.Status.Code, len(s.Attributes))
	}
}
```

```
$ go test -v -run TestLoanTrace ./internal/obs/
loans.Create   parent=cfbe8ed1c544bf19 statut=Error attributs=3
POST /loans    parent=0000000000000000 statut=Unset attributs=10
--- PASS: TestLoanTrace (0.00s)
```

- **L'arbre** : le span `loans.Create` a pour parent le span serveur `POST /loans`. `otelhttp` l'a nommé
  d'après le motif.
- **Le span de dépôt porte l'opération** (`INSERT`) et la table, jamais la requête SQL avec ses valeurs, ni
  l'e-mail reçu en paramètre. Le test vérifie qu'aucun attribut ne contient `@`.
- **`RecordError` et `SetStatus(codes.Error, …)`** marquent l'échec dans la trace.
- **Le span serveur reste `Unset`** pour une réponse 409 : `otelhttp` ne marque `Error` que les 5xx d'un
  serveur. Un 4xx est une erreur **du client**, pas du service. C'est la même distinction que pour le taux
  d'erreurs de RED.
- **`tracetest.NewInMemoryExporter`** avec `WithSyncer` rend les spans disponibles dès leur fin. Un piège
  trouvé en écrivant E4 : `Shutdown` **efface** les spans de l'exportateur en mémoire. Il faut les lire
  avant.

Pour les questions 1 à 3 et 5 : le middleware de métriques du cours, des compteurs métier
(`loans_created_total`, `loans_refused_total{reason}` avec trois ou quatre motifs fixes), `otelhttp` en
entrée et sur les clients, `TraceHandler` sur le logger. Le coût (question 5) se compare au tableau du cours :
de l'ordre de 10 µs par requête, à rapporter à la durée de la route.

---

## Défi — Prometheus et Jaeger pour de vrai

```go
// Fichier : cmd/svc/main.go
// Commande svc : le catalogue ou le stock (ROLE), instrumentés, pour Prometheus et Jaeger.
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"syscall"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.43.0"

	"example.com/svc/internal/obs"
)

func main() {
	role := os.Getenv("ROLE")
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	log := slog.New(obs.TraceHandler{Handler: slog.NewJSONHandler(os.Stdout, nil)}).With("service", role)

	// L'exportateur OTLP lit OTEL_EXPORTER_OTLP_ENDPOINT (par exemple http://jaeger:4318).
	exp, err := otlptracehttp.New(ctx)
	if err != nil {
		log.Error("exportateur", "err", err)
		os.Exit(1)
	}
	tp := sdktrace.NewTracerProvider(sdktrace.WithBatcher(exp),
		sdktrace.WithResource(resource.NewSchemaless(semconv.ServiceName(role))),
		sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(1))))
	defer tp.Shutdown(context.Background())
	otel.SetTracerProvider(tp)

	m := obs.NewMetrics(role)
	mux := http.NewServeMux()
	client := &http.Client{Transport: otelhttp.NewTransport(http.DefaultTransport), Timeout: 2 * time.Second}
	switch role {
	case "stock":
		mux.HandleFunc("GET /stock/{id}", func(w http.ResponseWriter, r *http.Request) {
			if id, _ := strconv.Atoi(r.PathValue("id")); id%10 == 0 {
				time.Sleep(300 * time.Millisecond) // un livre sur dix est lent
				log.WarnContext(r.Context(), "stock lent", "book_id", id)
			}
			fmt.Fprintln(w, 3)
		})
	case "catalogue":
		stockURL := os.Getenv("STOCK_URL")
		mux.HandleFunc("GET /books/{id}", func(w http.ResponseWriter, r *http.Request) {
			req, _ := http.NewRequestWithContext(r.Context(), "GET", stockURL+"/stock/"+r.PathValue("id"), nil)
			resp, err := client.Do(req)
			if err != nil {
				log.ErrorContext(r.Context(), "stock injoignable", "err", err)
				http.Error(w, "stock indisponible", http.StatusBadGateway)
				return
			}
			resp.Body.Close()
			fmt.Fprintln(w, "ok")
		})
	}
	admin := http.NewServeMux()
	admin.Handle("GET /metrics", m.Handler())
	go http.ListenAndServe(":6060", admin)
	srv := &http.Server{Addr: ":8080", Handler: otelhttp.NewHandler(m.Middleware(mux), role)}
	go srv.ListenAndServe()
	log.Info("démarrage")
	<-ctx.Done()
	sctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	srv.Shutdown(sctx)
}
```

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.27.3 AS build
WORKDIR /src
COPY go.* ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /out/svc ./cmd/svc

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/svc /svc
ENTRYPOINT ["/svc"]
```

```yaml
# compose.yaml
services:
  catalogue:
    build: .
    environment:
      ROLE: catalogue
      STOCK_URL: http://stock:8080
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4318
    ports: ["127.0.0.1:18080:8080"]
  stock:
    build: .
    environment:
      ROLE: stock
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4318
  prometheus:
    image: prom/prometheus:v3.5.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts.yml:/etc/prometheus/alerts.yml:ro
    ports: ["127.0.0.1:19090:9090"]
  jaeger:
    image: jaegertracing/jaeger:2.9.0
    ports: ["127.0.0.1:16686:16686"]
```

```yaml
# prometheus.yml
global:
  scrape_interval: 5s
  evaluation_interval: 5s

rule_files: [/etc/prometheus/alerts.yml]

scrape_configs:
  - job_name: services
    static_configs:
      - targets: ["catalogue:6060", "stock:6060"]
```

```yaml
# alerts.yml
groups:
  - name: latence
    rules:
      - alert: CatalogueLent
        expr: histogram_quantile(0.99, sum by (le, route) (rate(http_request_duration_seconds_bucket{service="catalogue"}[1m]))) > 0.25
        for: 30s
        labels:
          severity: page
        annotations:
          summary: "p99 de {{ $labels.route }} au-dessus de 250 ms"
```

```
$ docker compose up -d --build
$ for i in $(seq 1 3000); do curl -s -o /dev/null http://127.0.0.1:18080/books/$i; done &
$ curl -s http://127.0.0.1:19090/api/v1/targets      # (résumé)
catalogue:6060 up
stock:6060 up
```

Les requêtes PromQL, exécutées par l'API de Prometheus (`/api/v1/query`) après 75 s de charge :

```
# sum by (service, route) (rate(http_requests_total[1m]))
{'route': 'GET /stock/{id}', 'service': 'stock'} 23.2727
{'route': 'GET /books/{id}', 'service': 'catalogue'} 23.2727

# taux d'erreurs : (5xx or total * 0) / total
{'service': 'stock'} 0
{'service': 'catalogue'} 0

# histogram_quantile(0.99, sum by (le, service, route) (rate(http_request_duration_seconds_bucket[1m])))
{'route': 'GET /stock/{id}', 'service': 'stock'} 0.475
{'route': 'GET /books/{id}', 'service': 'catalogue'} 0.475

# le même avec 0.5
{'route': 'GET /stock/{id}', 'service': 'stock'} 0.0028
{'route': 'GET /books/{id}', 'service': 'catalogue'} 0.0028

# /api/v1/alerts
CatalogueLent firing p99 de GET /books/{id} au-dessus de 250 ms
```

- **Le débit** est de 23 requêtes par seconde : la boucle `curl` est séquentielle, et un livre sur dix
  attend 300 ms. C'est la loi de Little (N11 L7) côté générateur.
- **Le taux d'erreurs** : la première version, `sum(rate(…{status=~"5.."}…)) / sum(rate(…))`, renvoyait
  un résultat **vide**. Sans aucune réponse 5xx, le numérateur n'a aucune série, et la division non plus. Une
  alerte écrite ainsi ne se déclencherait jamais, et un tableau de bord afficherait « pas de données » au lieu
  de 0. `or … * 0` fournit une série à zéro.
- **Le p99 vaut 0,475 s**, alors que les requêtes lentes durent 300 ms : c'est l'erreur de E2. Les
  observations sont entre les bornes 0,25 et 0,5, et l'interpolation les suppose réparties dans tout
  l'intervalle. L'alerte se déclenche à raison (le vrai p99 est bien au-dessus de 250 ms), mais la **valeur**
  affichée est fausse. Une borne à 0,3 ou 0,35 la corrigerait. Le p50 (2,8 ms) est précis, car les bornes
  sont serrées en bas.
- **L'alerte** est passée à `firing` après ses 30 s de `for:`. Elle ne sonne pas sur un pic isolé.

Une trace lente, trouvée par l'API de Jaeger (`/api/traces?service=catalogue&minDuration=250ms&limit=1`) :

```
trace b8b42e5812a803dd5bdaf863cad13601
  catalogue GET /books/{id}      301.7 ms
  catalogue HTTP GET             301.5 ms
  stock     GET /stock/{id}      300.8 ms
```

Puis les journaux de **cette** trace, dans les deux services (`docker compose logs | grep b8b42e58…`) :

```
{"level": "WARN", "msg": "stock lent", "service": "stock", "book_id": 1820, "trace_id": "b8b42e5812a803dd5bdaf863cad13601", "span_id": "36722dd0f92f3cb3"}
```

On a parcouru le chemin du cours de bout en bout :
- l'**alerte** sur une métrique (p99 du catalogue) ;
- une **trace** lente, où le temps est passé dans le stock ;
- les **journaux** de cette trace, avec la raison : le stock est lent pour le livre 1820.

Le catalogue n'a écrit aucune ligne pour cette requête, qui n'a pas échoué. Sa ligne de requête, si on
l'ajoutait (leçon 6), porterait le même `trace_id`.

Les images utilisées : `prom/prometheus:v3.5.0` et `jaegertracing/jaeger:2.9.0`, épinglées comme les
actions de la leçon 4.

---

## Quiz

1. Prometheus **interroge** périodiquement l'URL `/metrics` de chaque instance, qui répond par l'état
   courant de toutes ses séries, en texte.
2. Compteur : un total qui croît (requêtes, erreurs). Jauge : une valeur qui monte et descend (requêtes en
   cours, taille de file). Histogramme : une distribution (durées, tailles), pour calculer des percentiles.
3. Le taux se calcule à partir de deux relevés du compteur (`rate`), sur la fenêtre de son choix, de façon
   robuste aux relevés manqués et aux redémarrages. Un taux exposé est figé sur une fenêtre choisie par le
   service, et il ne s'agrège pas.
4. Le percentile est interpolé dans l'intervalle qui contient le rang : si les bornes sont larges autour des
   valeurs réelles, l'erreur est grande (+89 ms dans E2, 475 ms au lieu de ~300 ms dans le défi).
5. Les compteurs d'histogramme s'additionnent entre instances, puis on calcule le percentile global. Les
   percentiles d'un *summary*, calculés dans chaque instance, ne se combinent pas (N11 L7).
6. Il est testable, et il ne contient que ce que le service y a mis, sans métriques ajoutées à notre insu
   par une bibliothèque. On y ajoute explicitement les collecteurs Go et processus.
7. Une opération, avec un nom, un début, une fin, des attributs, un statut, l'identifiant de la trace et
   celui de son parent. D'un service à l'autre, l'en-tête `traceparent` transmet l'identifiant de trace et
   celui du span appelant, qui devient le parent.
8. Pour que la décision d'échantillonnage prise à l'entrée soit suivie partout : des traces complètes, sans
   spans orphelins.
9. L'export se fait par lots, en arrière-plan. `Shutdown` envoie les spans en attente : sans lui, les
   derniers sont perdus à l'arrêt.
10. Par le `trace_id` : il est dans la trace, dans chaque ligne de journal (`TraceHandler`), et on part d'une
    alerte sur une métrique pour trouver une trace de la route concernée.
