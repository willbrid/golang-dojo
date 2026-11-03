# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le code des exercices faciles, ajouté au paquet `logx` :

```go
// Fichier : internal/logx/fix.go
package logx

import (
	"context"
	"log/slog"
	"runtime"
	"strings"
	"testing"
	"time"
)

// E2 : une fonction utilitaire qui journalise une erreur, et qui doit indiquer la ligne de SON
// appelant, pas la sienne.
func LogError(ctx context.Context, log *slog.Logger, msg string, err error) {
	if !log.Enabled(ctx, slog.LevelError) {
		return
	}
	var pcs [1]uintptr
	runtime.Callers(2, pcs[:]) // 0 = Callers, 1 = LogError, 2 = l'appelant
	r := slog.NewRecord(time.Now(), slog.LevelError, msg, pcs[0])
	r.AddAttrs(slog.Any("err", err))
	_ = log.Handler().Handle(ctx, r)
}

// E5 : un io.Writer qui envoie chaque ligne à t.Log. Le test ne montre les journaux qu'en cas
// d'échec (ou avec -v).
type TestWriter struct{ T testing.TB }

func (w TestWriter) Write(p []byte) (int, error) {
	w.T.Helper()
	w.T.Log(strings.TrimSuffix(string(p), "\n"))
	return len(p), nil
}
```

```go
// Fichier : internal/logx/fix_test.go
package logx

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"log/slog"
	"strings"
	"testing"
)

func TestE1(t *testing.T) {
	var buf bytes.Buffer
	log := slog.New(slog.NewTextHandler(&buf, &slog.HandlerOptions{ReplaceAttr: dropTime}))
	id, cmdID, err := 42, "C-17", errors.New("délai dépassé")
	n, max, rate, attempt := 812, 1000, 93.4, 2
	log.Info("connexion", "user_id", id) // pas l'adresse IP : donnée personnelle, à journaliser seulement si nécessaire
	log.Error("lecture de commande impossible", "command_id", cmdID, "err", err)
	log.Debug("état du cache", "entries", n, "capacity", max, "hit_rate", rate)
	log.Warn("relance", "service", "paiement", "attempt", attempt, "max_attempts", 3)
	fmt.Print(buf.String())
}

func dropTime(groups []string, a slog.Attr) slog.Attr {
	if len(groups) == 0 && a.Key == slog.TimeKey {
		return slog.Attr{} // un attribut vide n'est pas écrit
	}
	return a
}

func helper(log *slog.Logger, err error) { log.Error("échec", "err", err) } // naïf

func TestE2Source(t *testing.T) {
	var buf bytes.Buffer
	log := slog.New(slog.NewTextHandler(&buf, &slog.HandlerOptions{AddSource: true, ReplaceAttr: dropTime}))
	err := errors.New("boum")
	helper(log, err)                                  // ligne : celle de helper
	LogError(context.Background(), log, "échec", err) // ligne : celle-ci
	fmt.Print(buf.String())
}

func BenchmarkSource(b *testing.B) {
	for _, src := range []bool{false, true} {
		log := slog.New(slog.NewJSONHandler(sink{}, &slog.HandlerOptions{AddSource: src}))
		b.Run(fmt.Sprintf("AddSource=%v", src), func(b *testing.B) {
			for b.Loop() {
				log.Info("requête", "method", "GET", "status", 200)
			}
		})
	}
}

func TestE3Groups(t *testing.T) {
	var buf bytes.Buffer
	var seen []string
	opts := &slog.HandlerOptions{ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
		if a.Key == "token" {
			seen = append(seen, fmt.Sprintf("groupes=%v clé=%s", groups, a.Key))
		}
		return dropTime(groups, redact(groups, a))
	}}
	log := slog.New(slog.NewJSONHandler(&buf, opts))
	log.Info("appel", slog.Group("http", "method", "GET", "status", 200, "token", "t1"), "après", 1)
	log.WithGroup("http").Info("appel", "method", "GET", "status", 200, "token", "t1")
	fmt.Print(buf.String())
	fmt.Println(strings.Join(seen, "\n"))
}

type tokenErr struct{ token string }

func (e tokenErr) Error() string { return "jeton refusé : " + e.token }

func TestE4Bypass(t *testing.T) {
	var buf bytes.Buffer
	log := New(&buf, slog.LevelInfo, false)
	tok := "t0k3n-XYZ"
	log.Info("appel", "valeur", tok)                   // 1. une autre clé
	log.Info("appel avec le jeton " + tok)             // 2. dans le message
	log.Error("échec", "err", tokenErr{tok})           // 3. dans le texte d'une erreur
	log.Info("requête", "url", "/export?api_key="+tok) // 4. dans une URL
	for _, l := range strings.Split(strings.TrimSpace(buf.String()), "\n") {
		fmt.Println(strings.Contains(l, tok), l[strings.Index(l, "msg="):])
	}
}

// E5
func TestE5(t *testing.T) {
	log := New(TestWriter{t}, slog.LevelInfo, false)
	log.Info("visible seulement avec -v ou si le test échoue")
}

func TestE5Late(t *testing.T) {
	go1, result := make(chan struct{}), make(chan any)
	t.Run("sous-test", func(t *testing.T) {
		log := New(TestWriter{t}, slog.LevelInfo, false)
		go func() { // une goroutine qui survit au sous-test
			<-go1
			defer func() { result <- recover() }()
			log.Info("trop tard")
		}()
	})
	close(go1) // le sous-test est terminé : la goroutine journalise maintenant
	fmt.Println("panique :", <-result)
}

// Et après la fin d'un test de premier niveau : la goroutine de TestE5LateA journalise pendant TestE5LateB.
var lateGo, lateResult = make(chan struct{}), make(chan any)

func TestE5LateA(t *testing.T) {
	log := New(TestWriter{t}, slog.LevelInfo, false)
	go func() {
		<-lateGo
		defer func() { lateResult <- recover() }()
		log.Info("bien trop tard")
	}()
}

func TestE5LateB(t *testing.T) {
	close(lateGo)
	fmt.Println("panique :", <-lateResult)
}

func TestE5Output(t *testing.T) {
	log := New(t.Output(), slog.LevelInfo, false) // Go 1.25 : pas d'emplacement ajouté
	log.Info("par t.Output")
}
```

## E1 — Réécrire en structuré

```
level=INFO msg=connexion user_id=42
level=ERROR msg="lecture de commande impossible" command_id=C-17 err="délai dépassé"
level=WARN msg=relance service=paiement attempt=2 max_attempts=3
--- PASS: TestE1 (0.00s)
```

(`dropTime` retire l'heure pour que la sortie soit lisible ; le `Debug` du cache n'apparaît pas au niveau
`INFO`.)

| Ligne | Niveau | Pourquoi |
|---|---|---|
| connexion | `INFO` | un événement normal du cycle de vie d'une session |
| lecture impossible | `ERROR` | un échec qui a empêché le travail demandé |
| état du cache | `DEBUG` | un détail de diagnostic ; en production, c'est le rôle d'une **métrique** (leçon 7) |
| relance | `WARN` | anormal, mais le service s'en sort ; à surveiller si cela devient fréquent |

L'adresse IP de la première ligne est une **donnée personnelle** dans de nombreux cadres juridiques. On ne
la journalise que si on en a besoin (sécurité, détection d'abus), avec une durée de conservation
définie. Le `user_id` suffit en général pour relier les lignes entre elles.

## E2 — La source

```
level=ERROR source=internal/logx/fix_test.go:32 msg=échec err=boum
level=ERROR source=internal/logx/fix_test.go:39 msg=échec err=boum
--- PASS: TestE2Source (0.00s)
```

Avec `AddSource: true`, chaque ligne porte `source=fichier:ligne`.

| | Temps | Allocations |
|---|---|---|
| `AddSource=false` | 910 ns | 0 |
| `AddSource=true` | 2 123 ns | 6 (584 o) |

L'option double le coût d'une ligne et alloue : il faut retrouver la fonction, le fichier et la ligne de
l'appelant à chaque appel.

**La fonction utilitaire.** La première ligne indique `fix_test.go:32`, la ligne **dans `helper`**, et non
celle de l'appelant. `slog` enregistre l'endroit où l'on appelle `log.Error`. La correction, dans
`LogError`, construit l'enregistrement elle-même :

- `runtime.Callers(2, pcs[:])` récupère l'adresse de l'**appelant** de `LogError` (0 = `Callers`,
  1 = `LogError`) ;
- `slog.NewRecord(…, pcs[0])` crée un enregistrement avec cette adresse comme source ;
- `log.Handler().Handle(ctx, r)` le transmet au gestionnaire.

La seconde ligne indique bien `fix_test.go:39`, la ligne de l'appel à `LogError`. Le test
`log.Enabled` évite tout ce travail quand le niveau est désactivé.

## E3 — Les groupes

```
{"level":"INFO","msg":"appel","http":{"method":"GET","status":200,"token":"[masqué]"},"après":1}
{"level":"INFO","msg":"appel","http":{"method":"GET","status":200,"token":"[masqué]"}}
groupes=[http] clé=token
groupes=[http] clé=token
--- PASS: TestE3Groups (0.00s)
```

Les deux formes donnent **le même JSON** pour les attributs du groupe. La différence est la portée :

- **`slog.Group("http", …)`** ne groupe que **ses** attributs. `"après":1` reste au niveau racine.
- **`log.WithGroup("http")`** place dans le groupe **tous** les attributs ajoutés ensuite, ceux de chaque
  appel et ceux des `With` suivants. C'est utile pour un composant qui reçoit un logger « déjà groupé ».

`ReplaceAttr` reçoit bien le chemin des groupes (`groupes=[http]`), et notre `redact` masque `token` dans
les deux cas. Il compare la clé seule. Pour masquer selon le contexte, par exemple `http.token` mais pas
`cache.token`, on s'appuierait sur `groups`.

## E4 — Le masquage qui ne suffit pas

```
true msg=appel valeur=t0k3n-XYZ
true msg="appel avec le jeton t0k3n-XYZ"
true msg=échec err="jeton refusé : t0k3n-XYZ"
true msg=requête url="/export?api_key=t0k3n-XYZ"
--- PASS: TestE4Bypass (0.00s)
```

`true` : le jeton figure dans la ligne. Les **quatre** contournements passent à travers `redact`.

| Contournement | Protection |
|---|---|
| une autre clé (`"valeur", tok`) | un type `Secret` (leçon 5) avec `LogValue`, qui protège quelle que soit la clé |
| dans le message (`"… " + tok`) | message **fixe** (cours) ; `go vet` ne le voit pas, la revue de code oui |
| dans le texte d'une erreur (`jeton refusé : t0k3n`) | une erreur qui ne recopie jamais le secret (leçon 5, `url.Parse`) ; au besoin `LogValue` sur le type d'erreur |
| dans une URL (`?api_key=…`) | journaliser le **motif** de route (`r.Pattern`), jamais l'URL brute ; ne pas passer de secret en paramètre d'URL |

`ReplaceAttr` est un filet de sécurité, pas une garantie. La vraie protection est que le secret ne **puisse**
pas s'afficher (`Secret`) et qu'on ne journalise jamais d'objets entiers.

## E5 — Les journaux dans les tests

```
    handler.go:321: time=2026-11-03T18:46:05.533+01:00 level=INFO msg="visible seulement avec -v ou si le test échoue"
--- PASS: TestE5 (0.00s)
=== NAME  TestE5Late
    fix.go:31: time=2026-11-03T18:46:05.534+01:00 level=INFO msg="trop tard"
panique : <nil>
--- PASS: TestE5Late (0.00s)
    --- PASS: TestE5Late/sous-test (0.00s)
--- PASS: TestE5LateA (0.00s)
panique : <nil>
--- PASS: TestE5LateB (0.00s)
    time=2026-11-03T18:46:05.534+01:00 level=INFO msg="par t.Output"
--- PASS: TestE5Output (0.00s)
```

- **Avec `t.Log`**, la ligne est préfixée par `handler.go:321`, une ligne **interne** de `slog`.
  `t.Helper()` dans `Write` ne suffit pas : les fonctions de `slog` entre notre code et `Write` ne sont pas
  marquées comme auxiliaires. L'emplacement affiché est donc trompeur.
- **`t.Output()`** (Go 1.25) écrit dans la sortie du test sans ajouter d'emplacement. C'est la bonne
  destination pour un logger de test : `New(t.Output(), …)`.
- **Après la fin d'un sous-test** (`TestE5Late`), la ligne « trop tard » n'est pas perdue : elle est
  rattachée au **test parent** (`=== NAME  TestE5Late`). Il n'y a pas de panique (`recover()` renvoie `nil`).
- **Après la fin d'un test de premier niveau** (`TestE5LateA`, la goroutine écrit pendant `TestE5LateB`),
  la ligne « bien trop tard » n'apparaît **nulle part**, et il n'y a toujours pas de panique.

Ce comportement a été mesuré sur go1.27.3 : on lit souvent que `t.Log` après la fin d'un test provoque une
panique, et ce n'est pas ce qu'on observe ici. Dans les deux cas, la vraie faute est la goroutine qui
survit au test : `synctest` (N6) ou un `WaitGroup` attendu dans `t.Cleanup` la feraient apparaître.

---

## Exercice intermédiaire — Les journaux du service de prêts

La partie vérifiable sans le service complet est la question 5 : une requête de bout en bout, dont on relit
toutes les lignes.

```go
// Fichier : internal/logx/request_test.go
package logx

import (
	"bufio"
	"bytes"
	"encoding/json"
	"errors"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

type borrower struct {
	ID    int
	Name  string
	Email string
}

func (b borrower) LogValue() slog.Value { return slog.GroupValue(slog.Int("id", b.ID)) }

// Intermédiaire, question 5 : une requête complète, puis on relit chaque ligne produite.
func TestRequestLines(t *testing.T) {
	var buf bytes.Buffer
	log := New(&buf, slog.LevelDebug, true)
	b := borrower{ID: 7, Name: "Ada Lovelace", Email: "ada@example.com"}

	mux := http.NewServeMux()
	mux.HandleFunc("POST /loans", func(w http.ResponseWriter, r *http.Request) {
		ctx := With(r.Context(), slog.Int("user_id", b.ID)) // posé par l'authentification
		log.DebugContext(ctx, "emprunteur chargé", "borrower", b)
		err := errors.New("livre déjà prêté")
		log.WarnContext(ctx, "prêt refusé", "err", err) // une fois, ici : le handler traite l'erreur
		http.Error(w, err.Error(), http.StatusConflict)
	})
	req := httptest.NewRequest("POST", "/loans", strings.NewReader(`{"password":"hunter2"}`))
	req.Header.Set("X-Request-ID", "req-123")
	Middleware(log, mux).ServeHTTP(httptest.NewRecorder(), req)

	sc := bufio.NewScanner(&buf)
	n := 0
	for sc.Scan() {
		n++
		line := sc.Text()
		var m map[string]any
		if err := json.Unmarshal([]byte(line), &m); err != nil {
			t.Fatalf("ligne non JSON : %s", line)
		}
		if m["request_id"] != "req-123" {
			t.Errorf("ligne sans identifiant de requête : %s", line)
		}
		for _, secret := range []string{"hunter2", "ada@example.com", "Lovelace"} {
			if strings.Contains(line, secret) {
				t.Errorf("donnée sensible %q : %s", secret, line)
			}
		}
		t.Log(line)
	}
	if n != 3 {
		t.Fatalf("%d lignes, attendu 3", n)
	}
}
```

```
$ go test -v -run TestRequestLines ./internal/logx/
    request_test.go:58: {"time":"…","level":"DEBUG","msg":"emprunteur chargé","borrower":{"id":7},"request_id":"req-123","user_id":7}
    request_test.go:58: {"time":"…","level":"WARN","msg":"prêt refusé","err":"livre déjà prêté","request_id":"req-123","user_id":7}
    request_test.go:58: {"time":"…","level":"INFO","msg":"requête","method":"POST","route":"POST /loans","status":409,"durée":…,"request_id":"req-123"}
--- PASS: TestRequestLines (0.00s)
```

- **Les trois lignes portent `request_id`**, y compris celles écrites par le handler, qui ne le connaît pas.
- **Ni le nom, ni l'e-mail, ni le mot de passe** du corps de la requête n'apparaissent. `borrower` ne donne
  que son `id`, et le corps n'est jamais journalisé.
- **Une observation : la ligne de requête n'a pas `user_id`.** Il est ajouté dans un contexte **enfant**,
  créé dans le handler. Le middleware, qui écrit sa ligne après `next.ServeHTTP`, ne voit que son propre
  contexte. Pour l'avoir dans la ligne de requête, le middleware d'authentification doit s'exécuter
  **avant** celui de journalisation et enrichir le contexte qu'il transmet. On peut aussi faire déposer
  l'information par le handler dans une structure mutable placée dans le contexte par le middleware.
- **3. Journaliser une fois.** L'erreur « livre déjà prêté » est journalisée dans le handler, qui la
  **traite** en répondant 409. Les couches service et dépôt la **renvoient** sans la journaliser (N2 L6).

Les questions 1 à 4 s'appliquent au code du service : `logx.New` dans `main` avec le format et le niveau de
la configuration, le middleware de N10 L10 remplacé par `logx.Middleware`, et `LogValue` sur l'emprunteur et
sur la configuration (déjà fait par le type `Secret` de la leçon 5).

---

## Défi — Échantillonner et limiter

```go
// Fichier : internal/logx/sample.go
package logx

import (
	"context"
	"fmt"
	"log/slog"
	"sync"
	"time"
)

// SampleHandler laisse passer, pour chaque couple (niveau, message), les First premières lignes de
// chaque fenêtre, puis une sur Every. Au début de la fenêtre suivante, il écrit une ligne de synthèse
// par message échantillonné.
type SampleHandler struct {
	slog.Handler
	s *sampler // partagé par les copies (WithAttrs) : une inondation reste une inondation
}

type key struct {
	level slog.Level
	msg   string
}

type sampler struct {
	first, every int
	window       time.Duration
	now          func() time.Time

	mu      sync.Mutex
	start   time.Time
	seen    map[key]int
	dropped map[key]int
}

func NewSampleHandler(h slog.Handler, first, every int, window time.Duration, now func() time.Time) SampleHandler {
	return SampleHandler{h, &sampler{first: first, every: every, window: window, now: now,
		seen: map[key]int{}, dropped: map[key]int{}}}
}

func (h SampleHandler) Handle(ctx context.Context, r slog.Record) error {
	k := key{r.Level, r.Message}
	keep, summary := h.s.decide(k)
	for _, s := range summary {
		_ = h.Handler.Handle(ctx, s)
	}
	if !keep {
		return nil
	}
	return h.Handler.Handle(ctx, r)
}

// decide met à jour les compteurs ; il renvoie les lignes de synthèse si une fenêtre se termine.
func (s *sampler) decide(k key) (bool, []slog.Record) {
	s.mu.Lock()
	defer s.mu.Unlock()
	now := s.now()
	var summary []slog.Record
	if now.Sub(s.start) >= s.window {
		for dk, n := range s.dropped {
			r := slog.NewRecord(now, slog.LevelWarn, "lignes supprimées par échantillonnage", 0)
			r.AddAttrs(slog.String("message", dk.msg), slog.String("niveau", dk.level.String()),
				slog.Int("supprimées", n), slog.String("fenêtre", fmt.Sprint(s.window)))
			summary = append(summary, r)
		}
		s.start, s.seen, s.dropped = now, map[key]int{}, map[key]int{}
	}
	s.seen[k]++
	n := s.seen[k]
	if n <= s.first || (n-s.first)%s.every == 0 {
		return true, summary
	}
	s.dropped[k]++
	return false, summary
}

func (h SampleHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return SampleHandler{h.Handler.WithAttrs(attrs), h.s}
}

func (h SampleHandler) WithGroup(name string) slog.Handler {
	return SampleHandler{h.Handler.WithGroup(name), h.s}
}
```

```go
// Fichier : internal/logx/sample_test.go
package logx

import (
	"bytes"
	"encoding/json"
	"fmt"
	"log/slog"
	"strings"
	"sync"
	"testing"
	"testing/slogtest"
	"time"
)

type fakeClock struct {
	mu sync.Mutex
	t  time.Time
}

func (c *fakeClock) Now() time.Time      { c.mu.Lock(); defer c.mu.Unlock(); return c.t }
func (c *fakeClock) Add(d time.Duration) { c.mu.Lock(); defer c.mu.Unlock(); c.t = c.t.Add(d) }

func TestSampleContract(t *testing.T) {
	var buf bytes.Buffer
	slogtest.Run(t, func(t *testing.T) slog.Handler {
		buf.Reset()
		// First très grand : rien n'est supprimé, on ne vérifie que le contrat.
		return NewSampleHandler(slog.NewJSONHandler(&buf, nil), 1<<30, 1, time.Second, time.Now)
	}, func(t *testing.T) map[string]any {
		var m map[string]any
		if err := json.Unmarshal(buf.Bytes(), &m); err != nil {
			t.Fatal(err)
		}
		return m
	})
}

func TestSampling(t *testing.T) {
	var buf bytes.Buffer
	clk := &fakeClock{t: time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC)}
	h := NewSampleHandler(slog.NewTextHandler(&buf, &slog.HandlerOptions{ReplaceAttr: dropTime}), 3, 1000, time.Second, clk.Now)
	log := slog.New(h)
	var wg sync.WaitGroup
	for range 8 { // sous -race : les compteurs sont partagés
		wg.Go(func() {
			for range 6250 {
				log.With("composant", "db").Error("lecture impossible")
			}
		})
	}
	wg.Wait()
	log.Info("autre message") // jamais touché par l'inondation de l'autre message
	clk.Add(time.Second)
	log.Info("fenêtre suivante")
	out := buf.String()
	fmt.Print(out)
	if n := strings.Count(out, `msg="lecture impossible"`); n != 3+49 {
		t.Errorf("%d lignes gardées, attendu 3 + 49", n)
	}
	if !strings.Contains(out, "supprimées=49948") {
		t.Error("synthèse absente ou fausse")
	}
}

func BenchmarkSample(b *testing.B) {
	nu := slog.New(slog.NewJSONHandler(sink{}, nil))
	sampled := slog.New(NewSampleHandler(slog.NewJSONHandler(sink{}, nil), 1<<30, 1, time.Second, time.Now))
	for name, log := range map[string]*slog.Logger{"nu": nu, "échantillonné": sampled} {
		b.Run(name, func(b *testing.B) {
			b.RunParallel(func(pb *testing.PB) {
				for pb.Next() {
					log.Info("requête", "status", 200)
				}
			})
		})
	}
}
```

```
$ go test -race -v -run 'TestSampleContract|TestSampling' ./internal/logx/
--- PASS: TestSampleContract (0.00s)
level=ERROR msg="lecture impossible" composant=db
…                                             (52 lignes « lecture impossible » au total)
level=INFO msg="autre message"
level=WARN msg="lignes supprimées par échantillonnage" message="lecture impossible" niveau=ERROR supprimées=49948 fenêtre=1s
level=INFO msg="fenêtre suivante"
--- PASS: TestSampling (0.36s)
```

- **Les comptes.** Sur 50 000 lignes identiques, on garde les 3 premières, puis une sur 1 000 : 3 + 49 = 52.
  Les 49 948 autres sont supprimées. « autre message » passe : l'échantillonnage est **par message**, une
  inondation ne fait pas taire le reste.
- **La synthèse** est écrite au début de la fenêtre suivante, par la première ligne qui arrive. Sans ligne
  suivante, elle n'est jamais écrite. Une goroutine avec un `time.Ticker` corrigerait cela, au prix d'un
  arrêt à gérer (N6).
- **2. Les copies partagent les compteurs.** `With("composant", "db")` crée une copie à chaque appel, et
  elles pointent toutes sur le même `sampler`. Sinon, chaque `With` repartirait de zéro, et l'inondation
  passerait entièrement. Le verrou rend ce partage sûr : le test passe sous `-race` avec 8 goroutines.
  `slogtest` valide le contrat (avec un seuil qui ne supprime rien).
- **3. Le coût**, avec 8 goroutines en parallèle :

  | | ns par ligne |
  |---|---|
  | gestionnaire nu | 227 |
  | échantillonné | 512 |

  Le verrou unique est un point de contention (N11 L5), et chaque ligne cherche dans une map. Pour un
  service à quelques milliers de lignes par seconde, c'est négligeable. Pour réduire le coût, on pourrait
  découper le verrou par message, ou ne compter qu'à partir du niveau `WARN`.
- **4. La règle.** On n'échantillonne **jamais** la **première** occurrence d'un message dans une fenêtre,
  ni la synthèse. Le principe : on garde la preuve de chaque **type** de problème et son **volume**, et on
  supprime seulement la répétition.
  - Les `ERROR` s'échantillonnent avec un seuil élevé (les N premières complètes).
  - Les lignes de requête `INFO` s'échantillonnent prudemment : elles servent à reconstituer ce qu'a vécu un
    utilisateur. Le taux et la latence se mesurent par des **métriques** (leçon 7), qui ne s'échantillonnent
    pas.
  - Ce qui touche à la sécurité (authentification refusée, accès interdit) ne s'échantillonne pas.

---

## Quiz

1. Pour pouvoir chercher, filtrer et compter. Le message fixe identifie l'événement, les attributs typés se
   filtrent (`status >= 500`). Une phrase formatée ne se lit qu'à l'œil.
2. Les paires (`"status", 200`) sont plus courtes à écrire ; une paire incomplète donne `!BADKEY`, et
   `go vet` le signale. `slog.Attr` (`slog.Int("status", 200)`) est typé à la compilation et un peu plus
   rapide (`LogAttrs`).
3. `ERROR` : un échec qui demande une action. `WARN` : anormal mais surmonté. `INFO` : le cycle de vie et une
   ligne par requête. `DEBUG` : le diagnostic, désactivé en production.
4. Pour qu'une panne donne une ligne, pas une par couche. On la journalise là où on la **traite** (le handler,
   le middleware), et on la renvoie sans la journaliser partout ailleurs.
5. Sur la sortie standard. La plateforme collecte et achemine.
6. Un middleware le range dans le contexte. Un gestionnaire enveloppant l'ajoute à chaque enregistrement, à
   condition qu'on journalise avec les variantes `…Context`.
7. `Logger.With` et `WithGroup` appellent `WithAttrs`/`WithGroup` du gestionnaire. Sans redéfinition, ce sont
   celles du gestionnaire enveloppé qui répondent, et elles renvoient un gestionnaire **sans** l'enveloppe.
8. `LogValuer` sur les types (la valeur décide de sa représentation) et `ReplaceAttr` (masquage par clé).
   Leurs limites : un secret dans le message, sous une autre clé, dans une erreur ou une URL passe à travers
   (E4).
9. Environ 15 ns. Mais les arguments sont évalués avant l'appel (`string(data)` alloue quand même). Un
   `LogValuer` n'est évalué que si la ligne est écrite.
10. Un paquet de la bibliothèque standard qui vérifie qu'un `slog.Handler` respecte le contrat de
    l'interface (groupes, `WithAttrs`, attributs vides, heure nulle…), à partir d'une fonction qui relit ce
    que le gestionnaire a écrit.
