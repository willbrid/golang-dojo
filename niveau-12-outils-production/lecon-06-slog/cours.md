# Leçon 6 — Logs structurés

> Un journal de production n'est pas lu ligne à ligne par un humain. Il est collecté, indexé et interrogé :
> « toutes les erreurs de la requête `7KQ2…` », « les requêtes de plus d'une seconde sur cette route depuis
> une heure ». Pour cela, chaque ligne doit être une **donnée structurée** (des paires clé-valeur), pas une
> phrase. Depuis Go 1.21, la bibliothèque standard fournit `log/slog`. Cette leçon montre comment s'en
> servir dans un service :
> - les niveaux, les attributs et les gestionnaires ;
> - le contexte de la requête dans chaque ligne ;
> - le masquage automatique des secrets ;
> - le coût, mesuré ;
> - et ce qu'on ne journalise jamais.

## Objectifs

- Écrire des journaux structurés avec `slog` : messages, attributs, groupes, niveaux.
- Choisir un format (texte en local, JSON en production) et configurer le gestionnaire.
- Ajouter automatiquement l'identifiant de requête à chaque ligne, par le contexte.
- Garantir qu'aucun secret ni aucune donnée personnelle n'est écrit, par `LogValuer` et `ReplaceAttr`.
- Connaître le coût d'une ligne de journal, et celui d'une ligne désactivée.
- Tester un gestionnaire et ce qu'il écrit.

## Explication

### Une ligne = un événement et ses attributs

```go
log.Info("requête", "method", "GET", "route", "GET /users/{id}", "status", 200)
```

```
time=2026-11-03T13:12:01.123+01:00 level=INFO msg=requête method=GET route="GET /users/{id}" status=200
{"time":"2026-11-03T13:12:01.123+01:00","level":"INFO","msg":"requête","method":"GET","route":"GET /users/{id}","status":200}
```

- **Le message est fixe.** On ne met pas les valeurs dedans : `"requête"`, pas `"requête GET /users/42 en
  12 ms"`. On cherche ensuite tous les événements `msg=requête`, et on filtre sur les **attributs**.
- **Les attributs sont typés** : `status` est un nombre en JSON, on peut donc filtrer `status >= 500`. On les
  passe en paires clé-valeur (`"status", 200`), ou sous forme de `slog.Attr` (`slog.Int("status", 200)`),
  plus sûre. Une paire incomplète (`log.Info("x", "status")`) produit une clé `!BADKEY`, et `go vet` le signale (« call to slog.Info missing a final value »).
- **`slog.Group("db", …)`** regroupe des attributs sous un préfixe : `db.host=… db.port=…` en texte, un objet
  en JSON.
- **`log.With("module", "prêts")`** renvoie un logger qui ajoute ces attributs à chaque ligne. C'est le logger
  qu'on passe à un composant (N10 L3).

### Les niveaux

`slog` a quatre niveaux, `DEBUG`, `INFO`, `WARN` et `ERROR`. Une ligne sous le niveau configuré n'est pas
écrite. Une politique simple :

| Niveau | Quand | Exemple |
|---|---|---|
| `ERROR` | quelque chose a échoué **et** demande une action, ou a été vu par un utilisateur | une requête en 500, une tâche abandonnée |
| `WARN` | anormal, mais le service s'en est sorti | une relance réussie, une configuration ignorée (leçon 5) |
| `INFO` | les événements du cycle de vie et **une** ligne par requête | démarrage, arrêt, requête terminée |
| `DEBUG` | le détail pour un diagnostic, désactivé en production | les étapes internes d'un calcul |

Une erreur se journalise **une fois**, là où on la traite (souvent la frontière : le handler, le
middleware), pas à chaque niveau qui la remonte. Le message « journaliser puis renvoyer » produit cinq lignes
pour une seule panne, et rend les journaux illisibles (N2 L6).

Le niveau du gestionnaire peut être un `*slog.LevelVar`, modifiable à chaud sans course (leçon 5, défi).

### Les gestionnaires

Un `*slog.Logger` construit des **enregistrements** (`slog.Record`) et les passe à un **gestionnaire**
(`slog.Handler`), qui les met en forme et les écrit.

- **`slog.NewTextHandler(w, opts)`** écrit en `clé=valeur`, lisible dans un terminal.
- **`slog.NewJSONHandler(w, opts)`** écrit un objet JSON par ligne, le format attendu par les systèmes de
  collecte.

Le service écrit sur la **sortie standard**. C'est la plateforme (Docker, Kubernetes, journald) qui collecte et
achemine, et non le service qui ouvre un fichier ou une connexion vers un serveur de journaux. Le format se
choisit par la configuration (leçon 5) : texte en local, JSON en production.

`slog.SetDefault(logger)` fait de ce logger celui des fonctions globales (`slog.Info`) **et** du vieux paquet
`log` : les bibliothèques qui utilisent `log.Printf` écrivent alors aussi au bon format.

### Le contexte de la requête

L'identifiant de requête (N10 L10) doit figurer dans **chaque** ligne écrite pendant la requête, y compris
dans les paquets qui ne le connaissent pas. On le range dans le **contexte**, et un gestionnaire qui enveloppe
le vrai gestionnaire l'ajoute à chaque enregistrement :

```go
func (h ContextHandler) Handle(ctx context.Context, r slog.Record) error {
	if attrs, ok := ctx.Value(ctxKey{}).([]slog.Attr); ok {
		r.AddAttrs(attrs...)
	}
	return h.Handler.Handle(ctx, r)
}
```

Le code de l'application n'a qu'une chose à faire : utiliser les variantes `…Context`
(`log.InfoContext(ctx, …)`, `log.ErrorContext(ctx, …)`), qui transmettent le contexte au gestionnaire.
Un `log.Info` sans contexte ne peut pas savoir à quelle requête il appartient.

Un gestionnaire maison doit respecter le contrat de `slog.Handler` : groupes, `WithAttrs`, attributs vides,
heure nulle… Le paquet `testing/slogtest` le vérifie par une batterie de tests prête à l'emploi.

### Ce qu'on ne journalise jamais

- **Les secrets** : mots de passe, jetons, clés, en-têtes `Authorization` et `Cookie`, chaînes de connexion.
- **Les données personnelles** au-delà du nécessaire : adresses e-mail, noms, adresses IP selon le contexte
  légal, contenus de formulaires. Un journal est copié, conservé et consulté par bien plus de personnes que la
  base de données.
- **Les corps de requête et de réponse** complets.
- **Les URL brutes**, quand le chemin ou la requête contiennent des identifiants ou des jetons
  (`?token=…`). On journalise le **motif** de la route (`r.Pattern`), comme pour les métriques (N10 L10).

Deux protections, complémentaires :

1. **`LogValuer`.** Un type décide lui-même de ce qu'il donne quand on le journalise. `User.LogValue()`
   ne renvoie que l'identifiant, et `Secret.LogValue()` (leçon 5) renvoie `[masqué]`. La protection voyage
   avec le type, quel que soit le logger.
2. **`ReplaceAttr`.** Une fonction des options du gestionnaire, appelée pour **chaque** attribut avant son
   écriture. Elle masque les clés sensibles (`password`, `token`, `authorization`…), même dans un groupe.
   C'est un filet de sécurité pour ce que `LogValuer` n'a pas couvert, par exemple une `string` passée sous la
   clé `password`.

Aucune des deux n'est parfaite : un secret journalisé sous la clé `"valeur"` passe à travers `ReplaceAttr`.
La revue de code et les tests restent nécessaires.

### Le coût

Mesuré dans l'exemple (une ligne de trois attributs, écrite dans un écrivain qui jette les octets) :

| Appel | Temps par ligne | Allocations |
|---|---|---|
| `log.Printf` | 444 ns | 0 |
| `slog`, gestionnaire texte | 1 158 ns | 0 |
| `slog`, gestionnaire JSON | 1 030 ns | 0 |
| `slog`, JSON, `LogAttrs` | 987 ns | 0 |
| `slog.Debug` sous le niveau `INFO` | **15,5 ns** | 0 |

- **Environ 1 µs par ligne.** À une ligne par requête, c'est négligeable devant le moindre accès réseau
  (niveau 11, Amdahl). À cent lignes par requête, il faut se poser la question du volume, avant celle de la
  vitesse.
- **Une ligne désactivée ne coûte presque rien** : 15 ns. Mais les **arguments**, eux, sont évalués avant
  l'appel. `log.Debug("x", "dump", string(data))` convertit `data` même si la ligne n'est pas écrite : deux
  allocations pour 64 Kio, mesurées. Un `LogValuer` n'est au contraire appelé **que si** la ligne est
  écrite. C'est la façon de rendre paresseux un attribut coûteux. On peut aussi tester
  `log.Enabled(ctx, slog.LevelDebug)` avant un calcul cher.
- **`LogAttrs`** évite la conversion des paires en attributs et gagne ici 4 %. C'est une optimisation pour
  un chemin chaud que le profil désigne, pas une règle.

Un piège de mesure, trouvé en écrivant ce tableau : avec `log.New(io.Discard, …)`, `log.Printf` prend 8,8 ns.
Le paquet `log` **reconnaît** `io.Discard` et ne formate plus rien. Les benchmarks utilisent donc un écrivain
neutre (`sink`).

## Exemple

Un paquet `logx` : construction du logger, gestionnaire de contexte, masquage, middleware. Et un petit
programme qui produit les lignes d'un vrai service.

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27
```

```go
// Fichier : internal/logx/logx.go
// Package logx : la journalisation du service, construite sur log/slog.
package logx

import (
	"context"
	"io"
	"log/slog"
	"strings"
)

// New construit le logger du service : JSON en production, texte en local ; niveau réglable
// à chaud (leçon 5) ; secrets masqués ; attributs du contexte ajoutés à chaque ligne.
func New(w io.Writer, level slog.Leveler, json bool) *slog.Logger {
	opts := &slog.HandlerOptions{Level: level, ReplaceAttr: redact}
	var h slog.Handler = slog.NewTextHandler(w, opts)
	if json {
		h = slog.NewJSONHandler(w, opts)
	}
	return slog.New(ContextHandler{h})
}

// Les clés dont la valeur n'est jamais écrite, quel que soit l'endroit où elles apparaissent.
var sensitive = []string{"password", "passwd", "secret", "token", "authorization", "cookie", "api_key"}

// redact est appelée par le gestionnaire pour chaque attribut, juste avant de l'écrire.
func redact(groups []string, a slog.Attr) slog.Attr {
	k := strings.ToLower(a.Key)
	for _, s := range sensitive {
		if strings.Contains(k, s) {
			return slog.String(a.Key, "[masqué]")
		}
	}
	return a
}

type ctxKey struct{}

// With range des attributs dans le contexte : ils seront ajoutés à toute ligne écrite
// avec ce contexte (InfoContext, ErrorContext…).
func With(ctx context.Context, attrs ...slog.Attr) context.Context {
	prev, _ := ctx.Value(ctxKey{}).([]slog.Attr)
	all := append(append([]slog.Attr(nil), prev...), attrs...) // copie : le contexte parent ne change pas
	return context.WithValue(ctx, ctxKey{}, all)
}

// ContextHandler enveloppe un gestionnaire et lui ajoute les attributs du contexte.
type ContextHandler struct{ slog.Handler }

func (h ContextHandler) Handle(ctx context.Context, r slog.Record) error {
	if attrs, ok := ctx.Value(ctxKey{}).([]slog.Attr); ok {
		r.AddAttrs(attrs...)
	}
	return h.Handler.Handle(ctx, r)
}

// WithAttrs et WithGroup doivent renvoyer un ContextHandler, sinon logger.With(…) perdrait
// l'enveloppe et les attributs du contexte disparaîtraient.
func (h ContextHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return ContextHandler{h.Handler.WithAttrs(attrs)}
}

func (h ContextHandler) WithGroup(name string) slog.Handler {
	return ContextHandler{h.Handler.WithGroup(name)}
}
```

```go
// Fichier : internal/logx/middleware.go
package logx

import (
	"crypto/rand"
	"log/slog"
	"net/http"
	"time"
)

type statusWriter struct {
	http.ResponseWriter
	status int
}

func (s *statusWriter) WriteHeader(code int) { s.status = code; s.ResponseWriter.WriteHeader(code) }

// Middleware : un identifiant de requête dans le contexte, et UNE ligne par requête (N10 L10).
func Middleware(log *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get("X-Request-ID")
		if id == "" || len(id) > 64 {
			id = rand.Text()[:16]
		}
		ctx := With(r.Context(), slog.String("request_id", id))
		r = r.WithContext(ctx)
		w.Header().Set("X-Request-ID", id)
		sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}
		start := time.Now()
		next.ServeHTTP(sw, r)
		level := slog.LevelInfo
		if sw.status >= 500 {
			level = slog.LevelError
		}
		log.LogAttrs(ctx, level, "requête",
			slog.String("method", r.Method),
			slog.String("route", r.Pattern), // le motif, pas le chemin : pas d'identifiants dans les journaux
			slog.Int("status", sw.status),
			slog.Duration("durée", time.Since(start)),
		)
	})
}
```

```go
// Fichier : internal/logx/logx_test.go
package logx

import (
	"bytes"
	"context"
	"encoding/json"
	"io"
	"log"
	"log/slog"
	"strings"
	"testing"
	"testing/slogtest"
)

// slogtest vérifie qu'un gestionnaire respecte le contrat de slog.Handler (groupes, With, attributs vides…).
func TestContextHandlerContract(t *testing.T) {
	var buf bytes.Buffer
	slogtest.Run(t, func(t *testing.T) slog.Handler {
		buf.Reset()
		return ContextHandler{slog.NewJSONHandler(&buf, nil)}
	}, func(t *testing.T) map[string]any {
		var m map[string]any
		if err := json.Unmarshal(buf.Bytes(), &m); err != nil {
			t.Fatal(err)
		}
		return m
	})
}

func TestContextAttrs(t *testing.T) {
	var buf bytes.Buffer
	log := New(&buf, slog.LevelInfo, true)
	ctx := With(context.Background(), slog.String("request_id", "r1"))
	log.With("module", "prêts").InfoContext(ctx, "ok") // With ne doit pas perdre l'enveloppe
	log.Info("sans contexte")
	lines := strings.Split(strings.TrimSpace(buf.String()), "\n")
	if !strings.Contains(lines[0], `"request_id":"r1"`) || !strings.Contains(lines[0], `"module":"prêts"`) {
		t.Errorf("ligne 1 : %s", lines[0])
	}
	if strings.Contains(lines[1], "request_id") {
		t.Errorf("ligne 2 : %s", lines[1])
	}
}

func TestRedact(t *testing.T) {
	var buf bytes.Buffer
	log := New(&buf, slog.LevelInfo, false)
	log.Info("x", "password", "hunter2", "Authorization", "Bearer abc",
		slog.Group("db", "api_token", "t0k3n"), "user", "ada")
	out := buf.String()
	for _, s := range []string{"hunter2", "Bearer abc", "t0k3n"} {
		if strings.Contains(out, s) {
			t.Errorf("secret visible : %s", out)
		}
	}
	if !strings.Contains(out, "user=ada") {
		t.Errorf("attribut ordinaire perdu : %s", out)
	}
}

// --- Le coût ---

type expensive struct{ calls *int }

func (e expensive) LogValue() slog.Value { *e.calls++; return slog.StringValue("cher") }

// sink jette les octets, mais n'est PAS io.Discard : le paquet log reconnaît io.Discard et ne
// formate alors plus rien, ce qui fausserait la comparaison.
type sink struct{}

func (sink) Write(p []byte) (int, error) { return len(p), nil }

func BenchmarkLog(b *testing.B) {
	std := log.New(sink{}, "", log.LstdFlags)
	text := slog.New(slog.NewTextHandler(sink{}, nil))
	jsn := slog.New(slog.NewJSONHandler(sink{}, nil))
	ctx := context.Background()
	b.Run("log.Printf", func(b *testing.B) {
		for b.Loop() {
			std.Printf("requête method=%s route=%q status=%d", "GET", "GET /users/{id}", 200)
		}
	})
	b.Run("slog-texte", func(b *testing.B) {
		for b.Loop() {
			text.Info("requête", "method", "GET", "route", "GET /users/{id}", "status", 200)
		}
	})
	b.Run("slog-json", func(b *testing.B) {
		for b.Loop() {
			jsn.Info("requête", "method", "GET", "route", "GET /users/{id}", "status", 200)
		}
	})
	b.Run("slog-json-LogAttrs", func(b *testing.B) {
		for b.Loop() {
			jsn.LogAttrs(ctx, slog.LevelInfo, "requête",
				slog.String("method", "GET"), slog.String("route", "GET /users/{id}"), slog.Int("status", 200))
		}
	})
	b.Run("log.Printf-io.Discard", func(b *testing.B) {
		d := log.New(io.Discard, "", log.LstdFlags)
		for b.Loop() {
			d.Printf("requête method=%s route=%q status=%d", "GET", "GET /users/{id}", 200)
		}
	})
	b.Run("debug-désactivé", func(b *testing.B) {
		for b.Loop() {
			jsn.Debug("requête", "method", "GET", "route", "GET /users/{id}", "status", 200)
		}
	})
}

func TestDisabledLevelCost(t *testing.T) {
	jsn := slog.New(slog.NewJSONHandler(io.Discard, nil))
	n := 0
	data := make([]byte, 1<<16)
	jsn.Debug("x", "dump", string(data)) // la conversion string(data) est faite AVANT l'appel
	jsn.Debug("x", "v", expensive{&n})   // LogValue n'est appelée que si la ligne est écrite
	if allocs := testing.AllocsPerRun(100, func() { jsn.Debug("x", "dump", string(data)) }); allocs < 1 {
		t.Errorf("allocations : %v", allocs)
	} else {
		t.Logf("Debug désactivé avec string(data) : %v allocation(s) par appel", allocs)
	}
	t.Logf("LogValue appelée %d fois pour un Debug désactivé", n)
	jsn.Info("x", "v", expensive{&n})
	t.Logf("LogValue appelée %d fois après un Info", n)
}
```

```go
// Fichier : cmd/svc/main.go
// Commande svc : un service qui journalise comme en production.
package main

import (
	"errors"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"os"

	"example.com/svc/internal/logx"
)

type User struct {
	ID       int
	Email    string
	Password string
}

// LogValue : ce qu'un User donne quand on le journalise. Ni l'adresse, ni le mot de passe.
func (u User) LogValue() slog.Value {
	return slog.GroupValue(slog.Int("id", u.ID))
}

func main() {
	var level slog.LevelVar // INFO par défaut ; leçon 5 : réglable à chaud
	log := logx.New(os.Stdout, &level, os.Getenv("LOG_FORMAT") == "json")
	slog.SetDefault(log) // le paquet log et slog.Info passent aussi par ce logger

	log.Info("démarrage", "version", "v1.3.0", "addr", ":8080")
	log.Debug("invisible au niveau INFO")

	mux := http.NewServeMux()
	mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
		u := User{ID: 42, Email: "ada@example.com", Password: "hunter2"}
		log.InfoContext(r.Context(), "utilisateur chargé", "user", u)
		if r.URL.Query().Get("panne") != "" {
			err := errors.New("base indisponible")
			log.ErrorContext(r.Context(), "lecture impossible", "err", err)
			http.Error(w, "erreur interne", http.StatusInternalServerError)
			return
		}
		w.Write([]byte("ok\n"))
	})
	h := logx.Middleware(log, mux)

	// Deux requêtes simulées, pour voir les lignes produites.
	for _, url := range []string{"/users/42", "/users/42?panne=1"} {
		req := httptest.NewRequest("GET", url, nil)
		h.ServeHTTP(httptest.NewRecorder(), req)
	}
	log.Info("connexion", "password", "hunter2", "db", slog.GroupValue(slog.String("api_token", "t0k3n")))
}
```

Sortie en texte (`go run ./cmd/svc`) :

```
time=2026-11-03T18:42:00.453+01:00 level=INFO msg=démarrage version=v1.3.0 addr=:8080
time=2026-11-03T18:42:00.454+01:00 level=INFO msg="utilisateur chargé" user.id=42 request_id=KCIJ5YPAAQPB4BAE
time=2026-11-03T18:42:00.454+01:00 level=INFO msg=requête method=GET route="GET /users/{id}" status=200 durée=90.89µs request_id=KCIJ5YPAAQPB4BAE
time=2026-11-03T18:42:00.454+01:00 level=INFO msg="utilisateur chargé" user.id=42 request_id=J7V7RMUWAAPIB25J
time=2026-11-03T18:42:00.454+01:00 level=ERROR msg="lecture impossible" err="base indisponible" request_id=J7V7RMUWAAPIB25J
time=2026-11-03T18:42:00.454+01:00 level=ERROR msg=requête method=GET route="GET /users/{id}" status=500 durée=49.459µs request_id=J7V7RMUWAAPIB25J
time=2026-11-03T18:42:00.454+01:00 level=INFO msg=connexion password=[masqué] db.api_token=[masqué]
```

Et en JSON (`LOG_FORMAT=json go run ./cmd/svc`) :

```
{"time":"2026-11-03T18:42:00.577296044+01:00","level":"INFO","msg":"démarrage","version":"v1.3.0","addr":":8080"}
{"time":"2026-11-03T18:42:00.577568141+01:00","level":"INFO","msg":"utilisateur chargé","user":{"id":42},"request_id":"75O2LJ52TM4DJUFW"}
{"time":"2026-11-03T18:42:00.577588538+01:00","level":"INFO","msg":"requête","method":"GET","route":"GET /users/{id}","status":200,"durée":23074,"request_id":"75O2LJ52TM4DJUFW"}
{"time":"2026-11-03T18:42:00.577606282+01:00","level":"INFO","msg":"utilisateur chargé","user":{"id":42},"request_id":"OWYCHT5XMECEBP7L"}
{"time":"2026-11-03T18:42:00.577613895+01:00","level":"ERROR","msg":"lecture impossible","err":"base indisponible","request_id":"OWYCHT5XMECEBP7L"}
{"time":"2026-11-03T18:42:00.577623147+01:00","level":"ERROR","msg":"requête","method":"GET","route":"GET /users/{id}","status":500,"durée":18101,"request_id":"OWYCHT5XMECEBP7L"}
{"time":"2026-11-03T18:42:00.577629737+01:00","level":"INFO","msg":"connexion","password":"[masqué]","db":{"api_token":"[masqué]"}}
```

Les tests et les benchmarks :

```
$ go test -v ./internal/logx/
--- PASS: TestContextHandlerContract (0.00s)
--- PASS: TestContextAttrs (0.00s)
--- PASS: TestRedact (0.00s)
    logx_test.go:121: Debug désactivé avec string(data) : 2 allocation(s) par appel
    logx_test.go:123: LogValue appelée 0 fois pour un Debug désactivé
    logx_test.go:125: LogValue appelée 1 fois après un Info
--- PASS: TestDisabledLevelCost (0.01s)
ok  	example.com/svc/internal/logx	0.011s
$ go test -run XXX -bench Log -benchmem -count 5 ./internal/logx/   # médianes
BenchmarkLog/log.Printf-8               444.6 ns/op    0 B/op   0 allocs/op
BenchmarkLog/slog-texte-8              1158   ns/op    0 B/op   0 allocs/op
BenchmarkLog/slog-json-8               1030   ns/op    0 B/op   0 allocs/op
BenchmarkLog/slog-json-LogAttrs-8       987.3 ns/op    0 B/op   0 allocs/op
BenchmarkLog/log.Printf-io.Discard-8      8.8 ns/op    0 B/op   0 allocs/op
BenchmarkLog/debug-désactivé-8           15.5 ns/op    0 B/op   0 allocs/op
```

## Explication du code

| Élément | Rôle |
|---|---|
| `New(w, level, json)` | Le seul endroit qui construit le logger : format, niveau (un `slog.Leveler`, donc un `LevelVar` réglable à chaud), masquage, contexte. |
| `ReplaceAttr: redact` | Appelée pour chaque attribut, avec les noms des groupes qui l'entourent. Elle compare la clé, en minuscules, à la liste des mots sensibles. |
| `With(ctx, attrs...)` | Range des attributs dans le contexte. La slice est **copiée** : un contexte enfant qui ajoute un attribut ne modifie pas celui du parent, partagé peut-être par d'autres goroutines. |
| `ContextHandler.Handle` | Ajoute les attributs du contexte à l'enregistrement, puis délègue. |
| `WithAttrs`, `WithGroup` | Sans elles, `log.With(…)` appellerait celles du gestionnaire **enveloppé**, renverrait un gestionnaire nu, et les attributs du contexte disparaîtraient. `TestContextAttrs` le vérifie. |
| `slogtest.Run` | Lance les tests de contrat de `slog.Handler` sur notre gestionnaire. Il faut lui fournir de quoi relire ce qui a été écrit (ici, décoder le JSON). |
| `log.LogAttrs(ctx, level, …)` | Le niveau est choisi à l'exécution (`ERROR` si le statut est ≥ 500), et les attributs typés évitent toute conversion. |
| `User.LogValue()` | Le type décide de sa représentation : seulement `id`. L'e-mail et le mot de passe n'apparaissent dans aucune ligne. |
| `slog.SetDefault(log)` | `slog.Info` et le paquet `log` utilisent désormais ce logger. |
| `sink` dans les benchmarks | Un écrivain qui jette les octets sans être `io.Discard`, que `log` court-circuite. |

## Erreurs fréquentes

1. **Des valeurs dans le message** (`log.Info(fmt.Sprintf("utilisateur %d créé", id))`) : impossible à
   filtrer ou à compter.
2. **Une erreur journalisée à chaque niveau** qui la remonte.
3. **`log.Info` au lieu de `log.InfoContext`** dans le code d'une requête : la ligne n'a pas d'identifiant.
4. **Un gestionnaire maison sans `WithAttrs`/`WithGroup`** : les attributs du contexte disparaissent dès
   qu'on utilise `log.With`.
5. **Journaliser un objet entier** (une struct utilisateur, une requête HTTP, une configuration) : secrets
   et données personnelles avec.
6. **Le chemin brut ou l'URL complète** dans le journal : identifiants et jetons.
7. **Un calcul coûteux en argument d'un `Debug`**, payé même quand la ligne n'est pas écrite.
8. **Écrire dans un fichier** depuis le service, avec sa propre rotation, au lieu de la sortie standard.
9. **`ERROR` pour tout ce qui sort de l'ordinaire** : les alertes fondées sur `ERROR` sonnent sans arrêt, et
   on cesse de les regarder.

## Bonnes pratiques Go

- `log/slog`, gestionnaire JSON en production, texte en local, sur la sortie standard.
- Un message fixe, des attributs typés ; `slog.Attr` dans les chemins chauds.
- Le logger construit dans `main` et passé aux composants (`log.With("module", …)`), pas de logger global
  caché dans les paquets.
- L'identifiant de requête dans le contexte, ajouté par un gestionnaire ; les variantes `…Context` partout.
- `LogValuer` sur les types qui contiennent des données sensibles ; `ReplaceAttr` comme filet de sécurité.
- Une ligne par requête au niveau `INFO`, une erreur journalisée une fois, `DEBUG` désactivé en production.
- `testing/slogtest` pour tout gestionnaire maison.

## Ce que je dois retenir

- Un journal structuré est une donnée : message fixe, attributs typés, une ligne par événement.
- `slog` : un `Logger`, des niveaux, des gestionnaires texte ou JSON ; `With` pour les attributs communs.
- Le contexte porte l'identifiant de requête ; un gestionnaire enveloppant l'ajoute à chaque ligne.
- Secrets et données personnelles : `LogValuer` sur les types, `ReplaceAttr` en filet, jamais d'objet
  entier.
- Environ 1 µs par ligne, 15 ns si elle est désactivée, mais les arguments sont toujours évalués.

---

➡️ [Exercices](exercices.md)
