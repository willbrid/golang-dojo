# Leçon 10 — Observabilité

> Un service en production est une boîte noire, sauf ce qu'on a décidé, à l'avance, de faire sortir. La
> question « pourquoi les emprunts sont-ils lents depuis 14 h ? » n'a de réponse que si le service mesure la
> durée de ses requêtes, par route, et que ses journaux permettent de suivre **une** requête à travers
> plusieurs services. Ce n'est pas un ajout de fin de projet : c'est une décision d'**architecture** —
> quels signaux, à quels endroits, avec quels identifiants. Les outils (`log/slog`, Prometheus,
> OpenTelemetry) sont l'objet du niveau 12 ; cette leçon décide de ce qu'ils devront faire, avec la seule
> bibliothèque standard.

## Objectifs

1. Distinguer les trois signaux — journaux, métriques, traces — et ce que chacun permet.
2. Instrumenter aux **frontières** (requêtes entrantes, appels sortants, messages), une fois.
3. Mesurer un service avec RED et une ressource avec USE.
4. Propager un **identifiant de corrélation** à travers les services.
5. Maîtriser la **cardinalité** des métriques.
6. Alerter sur des symptômes, avec des objectifs de service.

## Explication

### Trois signaux, trois questions

| Signal | Répond à | Forme | Coût |
|---|---|---|---|
| **Métriques** | « Est-ce que ça va ? Depuis quand ça se dégrade ? » | des nombres agrégés dans le temps (compteurs, histogrammes) | faible et **constant**, quel que soit le trafic |
| **Journaux** | « Que s'est-il passé pour **cette** requête ? » | un événement par ligne, avec ses détails | proportionnel au trafic |
| **Traces** | « Où le temps a-t-il été passé, de service en service ? » | l'arbre des appels d'une requête, avec leurs durées | élevé ; on échantillonne |

On les combine : une **métrique** déclenche l'alerte (« le taux d'erreurs dépasse 1 % »), les **journaux**
filtrés par route et par période montrent les erreurs, l'**identifiant** d'une requête fautive mène à sa
**trace**, qui montre que le temps est perdu dans l'appel au catalogue.

### Instrumenter aux frontières

Le code métier ne doit pas être truffé d'instrumentation. Les mesures utiles se prennent aux
**frontières**, là où une requête entre ou sort :

- **requêtes entrantes** : un middleware HTTP (N7 L3), un intercepteur gRPC (N7 L12), l'enveloppe d'un
  consommateur de messages ;
- **appels sortants** : un `http.RoundTripper` décorateur, une enveloppe autour du client de base, du
  courtier, du client résilient (leçon 9) ;
- **tâches de fond** : chaque exécution d'un relais (leçon 8) ou d'un travail planifié.

C'est pour cela que l'observabilité est une décision d'architecture : si les dépendances sont des
interfaces (leçon 4), un décorateur les instrumente sans les modifier ; si le contexte est propagé
partout (N6 L8), l'identifiant de corrélation suit la requête. Un code qui crée ses clients au milieu des
fonctions, ou qui ne propage pas son contexte, est très difficile à observer après coup.

Les événements **métier** (« emprunt refusé : trop d'emprunts ») méritent aussi d'être comptés : une
métrique `loans_refused_total{reason=…}` dit au métier ce que le service fait réellement.

### RED et USE

Deux grilles pour ne rien oublier :

- **RED**, pour chaque service et chaque route : **R**ate (requêtes par seconde), **E**rrors (part des
  erreurs), **D**uration (distribution des durées — un histogramme, pas une moyenne).
- **USE**, pour chaque ressource (pool de connexions, file, processeur) : **U**tilization (part occupée),
  **S**aturation (travail en attente), **E**rrors. Le pool de connexions de N8 L5 (`InUse`, `WaitCount`)
  et l'âge de l'*outbox* (leçon 8) en sont des exemples.

Une **moyenne** de durée cache tout : dans l'exemple, la moyenne vaut 15 ms alors qu'une requête sur trois a
pris 40 ms. Les histogrammes (nombre de requêtes par tranche de durée) permettent de calculer les
**percentiles** (le p99 : la durée sous laquelle se trouvent 99 % des requêtes), qui seuls décrivent
l'expérience des utilisateurs les moins chanceux. Le niveau 11 y revient.

### L'identifiant de corrélation

Chaque requête reçoit un identifiant à l'entrée (repris de l'appelant s'il en fournit un, borné en taille
— projet 3, D5), mis dans le contexte, écrit dans **chaque** ligne de journal et **propagé** dans les appels
sortants (en-tête `X-Request-ID`). On peut alors retrouver tous les journaux d'une requête dans tous les
services. Les traces généralisent cette idée : le standard W3C *Trace Context* (en-tête `traceparent`),
qu'OpenTelemetry implémente (niveau 12), propage un identifiant de trace **et** la position de chaque appel
dans l'arbre.

### La cardinalité

Chaque **combinaison distincte** de valeurs d'étiquettes crée une **série** à stocker. Étiqueter par le
chemin brut (`/books/1`, `/books/2`…) crée une série par livre : mesuré, **1 000 séries** pour 1 000 livres,
contre **1** avec le motif de route (`GET /books/{id}`). Avec des millions d'identifiants, le système de
métriques s'effondre. Règles :

- étiquettes à **faible cardinalité** seulement : route (motif), méthode, statut, nom de dépendance ;
- jamais d'identifiant d'utilisateur, d'adresse IP, d'URL complète, de message d'erreur libre ;
- les détails par requête vont dans les **journaux** et les **traces**, pas dans les métriques.

Depuis Go 1.23, `r.Pattern` donne le motif de la route choisie par le `ServeMux` — y compris dans un
middleware placé **autour** du routeur, une fois `ServeHTTP` revenu (vérifié). Une route inconnue (404 du
routeur) laisse le motif vide : on la regroupe sous une seule valeur.

### Alerter sur les symptômes

On alerte sur ce que **voient les utilisateurs** — taux d'erreurs, durée au p99 — pas sur les causes
(processeur à 80 %, mémoire), qui ne sont souvent pas des problèmes. Un **objectif de service** (SLO) chiffre
l'attente : « 99,5 % des emprunts réussissent en moins de 300 ms, sur 30 jours ». L'écart toléré (0,5 %)
est le **budget d'erreurs** : tant qu'il n'est pas épuisé, on déploie ; quand il l'est, on stabilise.

### Quand ne pas le faire

- **Tout journaliser** : chaque appel de fonction, chaque valeur. Le volume coûte (stockage, temps), noie
  l'information, et finit par contenir des données personnelles ou des secrets (N7 L9, niveau 12).
- **Des métriques par utilisateur ou par identifiant** : la cardinalité explose.
- **Des traces distribuées pour un service unique et simple** : des métriques RED et des journaux corrélés
  suffisent.
- **Des tableaux de bord sans alerte** et des alertes sans action possible : une alerte doit dire quoi faire.

## Exemple

Un service « prêt » qui appelle un service « catalogue » : métriques RED par motif de route, une ligne de
journal par requête, identifiant propagé à l'appel sortant, et une démonstration de la cardinalité. Une
horloge simulée rend les durées déterministes ; la sortie est vérifiée par `go test`.

```go
// Fichier : obs/metrics.go
// Package obs : de l'observabilité avec la seule bibliothèque standard.
// (Le niveau 12 remplace ce registre par Prometheus et OpenTelemetry ; les DÉCISIONS restent.)
package obs

import (
	"fmt"
	"io"
	"slices"
	"strings"
	"sync"
	"time"
)

// Buckets : bornes supérieures des tranches de durée (la dernière, +Inf, est implicite).
var Buckets = []time.Duration{5 * time.Millisecond, 25 * time.Millisecond, 100 * time.Millisecond, 500 * time.Millisecond}

type histogram struct {
	counts []int64 // une case par borne, plus +Inf
	sum    time.Duration
	n      int64
}

type Registry struct {
	mu       sync.Mutex
	counters map[string]int64
	hists    map[string]*histogram
}

func NewRegistry() *Registry {
	return &Registry{counters: map[string]int64{}, hists: map[string]*histogram{}}
}

func key(name string, labels ...string) string {
	if len(labels) == 0 {
		return name
	}
	var b strings.Builder
	b.WriteString(name + "{")
	for i := 0; i+1 < len(labels); i += 2 {
		if i > 0 {
			b.WriteByte(',')
		}
		fmt.Fprintf(&b, "%s=%q", labels[i], labels[i+1])
	}
	b.WriteByte('}')
	return b.String()
}

func (r *Registry) Inc(name string, labels ...string) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.counters[key(name, labels...)]++
}

func (r *Registry) Observe(name string, d time.Duration, labels ...string) {
	r.mu.Lock()
	defer r.mu.Unlock()
	k := key(name, labels...)
	h, ok := r.hists[k]
	if !ok {
		h = &histogram{counts: make([]int64, len(Buckets)+1)}
		r.hists[k] = h
	}
	i, _ := slices.BinarySearch(Buckets, d)
	h.counts[i]++
	h.sum += d
	h.n++
}

// Series : le nombre de séries distinctes (compteurs + histogrammes) — leur coût de stockage.
func (r *Registry) Series() int {
	r.mu.Lock()
	defer r.mu.Unlock()
	return len(r.counters) + len(r.hists)
}

// WriteText écrit toutes les séries, triées, une par ligne.
func (r *Registry) WriteText(w io.Writer) {
	r.mu.Lock()
	defer r.mu.Unlock()
	var lines []string
	for k, v := range r.counters {
		lines = append(lines, fmt.Sprintf("%s %d", k, v))
	}
	for k, h := range r.hists {
		var parts []string
		for i, c := range h.counts {
			le := "+Inf"
			if i < len(Buckets) {
				le = Buckets[i].String()
			}
			parts = append(parts, fmt.Sprintf("≤%s:%d", le, c))
		}
		lines = append(lines, fmt.Sprintf("%s n=%d moyenne=%v [%s]", k, h.n, h.sum/time.Duration(h.n), strings.Join(parts, " ")))
	}
	slices.Sort(lines)
	for _, l := range lines {
		fmt.Fprintln(w, l)
	}
}
```

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
		if id == "" || len(id) > 64 {
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
// Fichier : obs/example_test.go
package obs_test

import (
	"fmt"
	"log"
	"net/http"
	"net/http/httptest"
	"os"
	"time"

	"exemple/obs"
)

func Example() {
	// Une horloge simulée qui avance de 3, 40, 3… ms : des durées déterministes.
	steps := []time.Duration{3, 40, 3, 600, 1, 1}
	cur, i := time.Date(2026, 3, 1, 0, 0, 0, 0, time.UTC), 0
	now := func() time.Time {
		t := cur
		if i%2 == 1 { // la fin de la requête
			cur = cur.Add(steps[i/2] * time.Millisecond)
			t = cur
		}
		i++
		return t
	}
	logger := log.New(os.Stdout, "", 0)
	reg := obs.NewRegistry()

	// Le service « catalogue », appelé par le service « prêt ».
	catalog := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("catalogue : reçu X-Request-ID =", r.Header.Get("X-Request-ID"))
	}))
	defer catalog.Close()
	client := &http.Client{Transport: obs.Transport{Base: http.DefaultTransport}}

	mux := http.NewServeMux()
	mux.HandleFunc("GET /books/{id}", func(w http.ResponseWriter, r *http.Request) {
		if r.PathValue("id") == "panne" {
			http.Error(w, "erreur interne", http.StatusInternalServerError)
			return
		}
		req, _ := http.NewRequestWithContext(r.Context(), "GET", catalog.URL, nil)
		if resp, err := client.Do(req); err == nil {
			resp.Body.Close()
		}
	})
	h := obs.Middleware(mux, reg, logger, now)

	for _, c := range []struct{ path, id string }{
		{"/books/1", "client-abc"}, {"/books/2", "client-def"}, {"/books/panne", "client-ghi"},
	} {
		req := httptest.NewRequest("GET", c.path, nil)
		req.Header.Set("X-Request-ID", c.id)
		h.ServeHTTP(httptest.NewRecorder(), req)
	}
	fmt.Println("--- /metrics")
	reg.WriteText(os.Stdout)

	// La cardinalité : 1 000 livres différents, deux façons d'étiqueter.
	byPath, byPattern := obs.NewRegistry(), obs.NewRegistry()
	for n := range 1000 {
		byPath.Inc("http_requests_total", "route", fmt.Sprintf("/books/%d", n))
		byPattern.Inc("http_requests_total", "route", "GET /books/{id}")
	}
	fmt.Println("séries, étiquette = chemin :", byPath.Series(), "; étiquette = motif :", byPattern.Series())
	// Output:
	// catalogue : reçu X-Request-ID = client-abc
	// requête id=client-abc route="GET /books/{id}" status=200 durée=3ms
	// catalogue : reçu X-Request-ID = client-def
	// requête id=client-def route="GET /books/{id}" status=200 durée=40ms
	// requête id=client-ghi route="GET /books/{id}" status=500 durée=3ms
	// --- /metrics
	// http_request_duration{route="GET /books/{id}"} n=3 moyenne=15.333333ms [≤5ms:2 ≤25ms:0 ≤100ms:1 ≤500ms:0 ≤+Inf:0]
	// http_requests_total{route="GET /books/{id}",status="200"} 2
	// http_requests_total{route="GET /books/{id}",status="500"} 1
	// séries, étiquette = chemin : 1000 ; étiquette = motif : 1
}
```
`go test -race ./...` passe.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Middleware` autour du `mux` | **Un** endroit instrumente toutes les routes : RED, journal, identifiant. |
| Identifiant repris ou créé, borné à 64 caractères | Corrélation avec l'appelant, sans laisser un client injecter 10 000 caractères dans les journaux. |
| `context.WithValue(…, ctxKey{}, id)` | L'identifiant suit la requête dans tout le code qui reçoit le contexte. |
| `statusWriter` | Capte le statut écrit par le handler, pour l'étiquette `status`. |
| `r.Pattern` après `ServeHTTP` | Le motif de route, rempli par le `ServeMux` ; `aucune` pour un 404 du routeur. |
| `http_requests_total{route, status}` | Le **R** et le **E** de RED. |
| `http_request_duration` en histogramme | Le **D** : 2 requêtes ≤ 5 ms, 1 ≤ 100 ms ; la moyenne (15 ms) ne dit rien de cette répartition. |
| `obs.Transport` | Décorateur de `RoundTripper` : l'identifiant part dans l'en-tête des appels sortants (le catalogue reçoit `client-abc`). |
| `r.Clone` dans `RoundTrip` | Un `RoundTripper` ne doit pas modifier la requête qu'on lui confie. |
| `Series()` : 1 000 contre 1 | La cardinalité, mesurée. |
| Horloge injectée | Des durées déterministes : l'instrumentation se teste comme le reste. |

## Erreurs fréquentes

1. **Instrumenter au milieu du code métier** plutôt qu'aux frontières.
2. **Des moyennes de durée** au lieu d'histogrammes et de percentiles.
3. **Des étiquettes à forte cardinalité** (chemin brut, identifiants, messages d'erreur).
4. **Un identifiant de corrélation non propagé** aux appels sortants, ou absent des journaux.
5. **Faire confiance à l'identifiant entrant** sans le borner.
6. **Alerter sur les causes** (processeur) plutôt que sur les symptômes (erreurs, latence).
7. **Journaliser les secrets** ou les données personnelles.
8. **Ajouter l'observabilité après coup**, dans un code sans contexte propagé ni dépendances injectées.

## Bonnes pratiques Go

- Middleware HTTP et décorateur de `RoundTripper` : instrumentation aux frontières, une fois.
- RED par route (motif `r.Pattern`) ; USE par ressource (pool, file, *outbox*).
- Histogrammes pour les durées ; percentiles pour les objectifs.
- Identifiant de corrélation dans le contexte, dans chaque ligne de journal, dans chaque appel sortant.
- Étiquettes à faible cardinalité ; le détail dans les journaux et les traces.
- Horloge injectée pour tester l'instrumentation ; outils standards au niveau 12.

## Ce que je dois retenir

- Métriques (est-ce que ça va ?), journaux (que s'est-il passé ?), traces (où est passé le temps ?).
- On instrumente aux frontières, grâce aux interfaces et au contexte : c'est une décision d'architecture.
- RED pour les services, USE pour les ressources ; histogrammes, pas moyennes.
- Un identifiant de corrélation, propagé partout.
- Cardinalité faible ; alertes sur les symptômes, chiffrées par des objectifs de service.

---

➡️ [Exercices](exercices.md)
