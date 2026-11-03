# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les exercices E1 à E4 et l'exercice intermédiaire modifient tous `Load`. Le corrigé donne donc **une**
version du paquet qui les intègre tous, puis le détail par exercice. Le défi ajoute `Holder` et le `main`.

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27
```

```go
// Fichier : internal/config/config.go
// Package config lit, valide et décrit la configuration du service.
//
// Précédence : valeurs par défaut < fichier (CONFIG_FILE) < environnement (préfixe) < options.
package config

import (
	"bytes"
	"encoding/json"
	"errors"
	"flag"
	"fmt"
	"io"
	"log/slog"
	"net/url"
	"os"
	"slices"
	"strconv"
	"strings"
	"time"
)

// Secret ne s'affiche jamais, ni par fmt (%v, %s, %#v), ni en JSON, ni dans slog.
type Secret string

func (Secret) String() string                  { return "[masqué]" }
func (Secret) GoString() string                { return `"[masqué]"` }
func (Secret) MarshalText() ([]byte, error)    { return []byte("[masqué]"), nil }
func (Secret) LogValue() slog.Value            { return slog.StringValue("[masqué]") }
func (s Secret) Reveal() string                { return string(s) } // le seul accès à la valeur
func (s Secret) Format(f fmt.State, verb rune) { io.WriteString(f, "[masqué]") }

type Config struct {
	Addr           string
	AdminAddr      string
	DatabaseURL    Secret
	APIToken       Secret
	ShutdownGrace  time.Duration
	RequestTimeout time.Duration
	LogLevel       slog.Level
	Workers        int
	Features       []string
}

type Env struct {
	Getenv   func(string) string
	Environ  func() []string // pour repérer les variables inconnues (E3)
	ReadFile func(string) ([]byte, error)
}

var OS = Env{Getenv: os.Getenv, Environ: os.Environ, ReadFile: os.ReadFile}

// value : une valeur qui se lit depuis du texte, utilisée pour l'environnement ET pour flag.
type value[T any] struct {
	p     *T
	parse func(string) (T, error)
}

func (v value[T]) Set(s string) error {
	x, err := v.parse(s)
	if err != nil {
		return err
	}
	*v.p = x
	return nil
}

func (v value[T]) String() string {
	if v.p == nil { // flag appelle String sur une valeur zéro pour l'aide
		return ""
	}
	return fmt.Sprint(*v.p)
}

func str(s string) (string, error) { return s, nil }

func level(s string) (slog.Level, error) {
	var l slog.Level
	err := l.UnmarshalText([]byte(s))
	return l, err
}

func list(s string) ([]string, error) {
	var out []string
	for f := range strings.SplitSeq(s, ",") {
		if f = strings.TrimSpace(f); f != "" {
			out = append(out, f)
		}
	}
	return out, nil
}

// setting décrit un réglage une fois : nom de variable, nom d'option, aide, valeur (E1).
type setting struct {
	env, flag, usage string
	v                flag.Value
}

func (c *Config) settings() []setting {
	return []setting{
		{"ADDR", "addr", "adresse d'écoute", value[string]{&c.Addr, str}},
		{"ADMIN_ADDR", "admin-addr", "adresse d'administration", value[string]{&c.AdminAddr, str}},
		{"SHUTDOWN_GRACE", "shutdown-grace", "délai d'arrêt propre", value[time.Duration]{&c.ShutdownGrace, time.ParseDuration}},
		{"REQUEST_TIMEOUT", "request-timeout", "délai maximal d'une requête", value[time.Duration]{&c.RequestTimeout, time.ParseDuration}},
		{"LOG_LEVEL", "log-level", "DEBUG, INFO, WARN ou ERROR", value[slog.Level]{&c.LogLevel, level}},
		{"WORKERS", "workers", "nombre de workers", value[int]{&c.Workers, strconv.Atoi}},
		{"FEATURES", "features", "fonctionnalités activées, séparées par des virgules", value[[]string]{&c.Features, list}},
	}
}

var secrets = []string{"DATABASE_URL", "API_TOKEN"} // jamais en option, jamais dans le fichier

func defaults() Config {
	return Config{Addr: ":8080", AdminAddr: "127.0.0.1:6060", ShutdownGrace: 10 * time.Second,
		RequestTimeout: 5 * time.Second, LogLevel: slog.LevelInfo, Workers: 4}
}

// Load construit la configuration. warn reçoit les avertissements (variables inconnues).
func Load(prefix string, args []string, env Env, out io.Writer, warn func(string)) (Config, error) {
	c := defaults()
	var errs []error
	bad := func(name string, err error) { errs = append(errs, fmt.Errorf("%s : %w", name, err)) }
	set := c.settings()

	// 1. Le fichier (intermédiaire).
	if p := env.Getenv(prefix + "CONFIG_FILE"); p != "" {
		if err := loadFile(&c, p, env.ReadFile); err != nil {
			bad(prefix+"CONFIG_FILE", err)
		}
	}
	// 2. L'environnement, avec préfixe (E2).
	for _, s := range set {
		if v := env.Getenv(prefix + s.env); v != "" {
			if err := s.v.Set(v); err != nil {
				bad(prefix+s.env, err)
			}
		}
	}
	for _, name := range secrets { // E4
		v, err := secret(env, prefix+name)
		if err != nil {
			bad(prefix+name, err)
		}
		switch name {
		case "DATABASE_URL":
			c.DatabaseURL = Secret(v)
		case "API_TOKEN":
			c.APIToken = Secret(v)
		}
	}
	// E3 : une variable au préfixe connu mais au nom inconnu est probablement une faute de frappe.
	known := []string{prefix + "CONFIG_FILE"}
	for _, s := range set {
		known = append(known, prefix+s.env)
	}
	for _, n := range secrets {
		known = append(known, prefix+n, prefix+n+"_FILE")
	}
	for _, kv := range env.Environ() {
		name, _, _ := strings.Cut(kv, "=")
		if strings.HasPrefix(name, prefix) && !slices.Contains(known, name) {
			warn(fmt.Sprintf("variable inconnue ignorée : %s", name))
		}
	}
	// 3. Les options.
	fs := flag.NewFlagSet("svc", flag.ContinueOnError)
	fs.SetOutput(out)
	for _, s := range set {
		fs.Var(s.v, s.flag, fmt.Sprintf("%s (%s%s)", s.usage, prefix, s.env))
	}
	if err := fs.Parse(args); err != nil {
		return Config{}, err
	}
	if fs.NArg() > 0 {
		bad("arguments", fmt.Errorf("inattendus : %q", fs.Args()))
	}
	// 4. La validation.
	errs = append(errs, c.validate(prefix)...)
	if len(errs) > 0 {
		return Config{}, fmt.Errorf("configuration invalide :\n%w", errors.Join(errs...))
	}
	return c, nil
}

func (c Config) validate(prefix string) []error {
	var errs []error
	bad := func(name string, err error) { errs = append(errs, fmt.Errorf("%s%s : %w", prefix, name, err)) }
	if c.DatabaseURL == "" {
		bad("DATABASE_URL", errors.New("obligatoire (ou DATABASE_URL_FILE)"))
	} else if u, err := url.Parse(c.DatabaseURL.Reveal()); err != nil {
		bad("DATABASE_URL", errors.New("adresse invalide"))
	} else if u.Scheme != "postgres" && u.Scheme != "postgresql" {
		bad("DATABASE_URL", fmt.Errorf("schéma %q, attendu postgres", u.Scheme))
	}
	inRange := func(name string, d, lo, hi time.Duration) {
		if d < lo || d > hi {
			bad(name, fmt.Errorf("%v hors de [%v, %v]", d, lo, hi))
		}
	}
	inRange("SHUTDOWN_GRACE", c.ShutdownGrace, time.Second, time.Minute)
	inRange("REQUEST_TIMEOUT", c.RequestTimeout, 100*time.Millisecond, time.Minute)
	if c.Workers < 1 || c.Workers > 256 {
		bad("WORKERS", fmt.Errorf("%d hors de [1, 256]", c.Workers))
	}
	if c.Addr == c.AdminAddr {
		bad("ADMIN_ADDR", errors.New("doit différer de ADDR"))
	}
	return errs
}

// secret lit NAME ou NAME_FILE, pas les deux (E4).
func secret(env Env, name string) (string, error) {
	v, p := env.Getenv(name), env.Getenv(name+"_FILE")
	switch {
	case v != "" && p != "":
		return "", fmt.Errorf("%s et %s_FILE sont toutes deux définies : en choisir une", name, name)
	case p != "":
		b, err := env.ReadFile(p)
		if err != nil {
			return "", err
		}
		return strings.TrimSpace(string(b)), nil
	}
	return v, nil
}

// Duration se lit "10s" en JSON (intermédiaire, question 3).
type Duration time.Duration

func (d *Duration) UnmarshalText(b []byte) error {
	x, err := time.ParseDuration(string(b))
	*d = Duration(x)
	return err
}

// fileConfig : des pointeurs, pour distinguer « absent » de « zéro ».
type fileConfig struct {
	Addr           *string     `json:"addr"`
	AdminAddr      *string     `json:"admin_addr"`
	ShutdownGrace  *Duration   `json:"shutdown_grace"`
	RequestTimeout *Duration   `json:"request_timeout"`
	LogLevel       *slog.Level `json:"log_level"`
	Workers        *int        `json:"workers"`
	Features       []string    `json:"features"`
	DatabaseURL    *string     `json:"database_url"` // déclaré pour être refusé explicitement
	APIToken       *string     `json:"api_token"`
}

func loadFile(c *Config, path string, read func(string) ([]byte, error)) error {
	b, err := read(path)
	if err != nil {
		return err
	}
	dec := json.NewDecoder(bytes.NewReader(b))
	dec.DisallowUnknownFields()
	var f fileConfig
	if err := dec.Decode(&f); err != nil {
		return err
	}
	if f.DatabaseURL != nil || f.APIToken != nil {
		return errors.New("un secret n'a rien à faire dans le fichier de configuration")
	}
	if f.Addr != nil {
		c.Addr = *f.Addr
	}
	if f.AdminAddr != nil {
		c.AdminAddr = *f.AdminAddr
	}
	if f.ShutdownGrace != nil {
		c.ShutdownGrace = time.Duration(*f.ShutdownGrace)
	}
	if f.RequestTimeout != nil {
		c.RequestTimeout = time.Duration(*f.RequestTimeout)
	}
	if f.LogLevel != nil {
		c.LogLevel = *f.LogLevel
	}
	if f.Workers != nil {
		c.Workers = *f.Workers
	}
	if f.Features != nil {
		c.Features = f.Features
	}
	return nil
}
```

```go
// Fichier : internal/config/holder.go
package config

import (
	"log/slog"
	"sync/atomic"
)

// Holder publie la configuration courante et la recharge à chaud (défi).
type Holder struct {
	cur   atomic.Pointer[Config]
	Level slog.LevelVar // à passer au gestionnaire de journaux : modifiable sans course
	load  func() (Config, error)
	log   *slog.Logger
}

func NewHolder(c Config, load func() (Config, error), log *slog.Logger) *Holder {
	h := &Holder{load: load, log: log}
	h.Level.Set(c.LogLevel)
	h.cur.Store(&c)
	return h
}

// Get renvoie la configuration courante. Elle ne doit pas être modifiée : on la remplace en entier.
func (h *Holder) Get() *Config { return h.cur.Load() }

// Reload relit la configuration. Invalide : on garde l'ancienne. Valide : seul le niveau de journal
// s'applique à chaud ; les autres changements attendent un redémarrage.
func (h *Holder) Reload() {
	next, err := h.load()
	if err != nil {
		h.log.Error("rechargement refusé, configuration inchangée", "err", err)
		return
	}
	old := h.Get()
	if next.Addr != old.Addr || next.AdminAddr != old.AdminAddr || next.Workers != old.Workers {
		h.log.Warn("réglages modifiés qui demandent un redémarrage : ignorés",
			"addr", next.Addr, "admin_addr", next.AdminAddr, "workers", next.Workers)
		next.Addr, next.AdminAddr, next.Workers = old.Addr, old.AdminAddr, old.Workers
	}
	if next.LogLevel != old.LogLevel {
		h.log.Info("niveau de journal modifié", "avant", old.LogLevel, "après", next.LogLevel)
	}
	h.Level.Set(next.LogLevel)
	h.cur.Store(&next)
}
```

```go
// Fichier : internal/config/config_test.go
package config

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"io/fs"
	"log/slog"
	"strings"
	"sync"
	"testing"
	"time"
)

func fakeEnv(vars, files map[string]string) Env {
	return Env{
		Getenv: func(k string) string { return vars[k] },
		Environ: func() []string {
			var out []string
			for k, v := range vars {
				out = append(out, k+"="+v)
			}
			return out
		},
		ReadFile: func(p string) ([]byte, error) {
			if s, ok := files[p]; ok {
				return []byte(s), nil
			}
			return nil, fs.ErrNotExist
		},
	}
}

const dsn = "postgres://app:s3cr3t@db/app"

func load(t *testing.T, vars, files map[string]string, args ...string) (Config, []string, error) {
	t.Helper()
	var warns []string
	c, err := Load("SVC_", args, fakeEnv(vars, files), io.Discard, func(m string) { warns = append(warns, m) })
	return c, warns, err
}

// Précédence complète : défauts < fichier < environnement < options.
func TestPrecedence(t *testing.T) {
	file := map[string]string{"/c.json": `{"workers": 6, "request_timeout": "3s"}`}
	for _, tc := range []struct {
		name    string
		vars    map[string]string
		args    []string
		workers int
		timeout time.Duration
	}{
		{"défauts", map[string]string{"SVC_DATABASE_URL": dsn}, nil, 4, 5 * time.Second},
		{"fichier", map[string]string{"SVC_DATABASE_URL": dsn, "SVC_CONFIG_FILE": "/c.json"}, nil, 6, 3 * time.Second},
		{"environnement > fichier", map[string]string{"SVC_DATABASE_URL": dsn, "SVC_CONFIG_FILE": "/c.json", "SVC_WORKERS": "16"}, nil, 16, 3 * time.Second},
		{"option > environnement", map[string]string{"SVC_DATABASE_URL": dsn, "SVC_CONFIG_FILE": "/c.json", "SVC_WORKERS": "16"}, []string{"-workers", "2", "-request-timeout", "1s"}, 2, time.Second},
	} {
		t.Run(tc.name, func(t *testing.T) {
			c, _, err := load(t, tc.vars, file, tc.args...)
			if err != nil {
				t.Fatal(err)
			}
			if c.Workers != tc.workers || c.RequestTimeout != tc.timeout {
				t.Fatalf("workers=%d timeout=%v, attendu %d et %v", c.Workers, c.RequestTimeout, tc.workers, tc.timeout)
			}
		})
	}
}

func TestErrors(t *testing.T) {
	for _, tc := range []struct {
		name  string
		vars  map[string]string
		files map[string]string
		want  string
	}{
		{"timeout hors plage", map[string]string{"SVC_DATABASE_URL": dsn, "SVC_REQUEST_TIMEOUT": "10ms"}, nil, "SVC_REQUEST_TIMEOUT : 10ms hors de [100ms, 1m0s]"},
		{"timeout invalide", map[string]string{"SVC_DATABASE_URL": dsn, "SVC_REQUEST_TIMEOUT": "vite"}, nil, "SVC_REQUEST_TIMEOUT : time: invalid duration"},
		{"secret en double", map[string]string{"SVC_DATABASE_URL": dsn, "SVC_DATABASE_URL_FILE": "/s"}, nil, "toutes deux définies"},
		{"champ inconnu", map[string]string{"SVC_DATABASE_URL": dsn, "SVC_CONFIG_FILE": "/c"}, map[string]string{"/c": `{"workres": 3}`}, `unknown field "workres"`},
		{"secret dans le fichier", map[string]string{"SVC_DATABASE_URL": dsn, "SVC_CONFIG_FILE": "/c"}, map[string]string{"/c": `{"database_url": "x"}`}, "un secret n'a rien à faire"},
	} {
		t.Run(tc.name, func(t *testing.T) {
			_, _, err := load(t, tc.vars, tc.files)
			if err == nil || !strings.Contains(err.Error(), tc.want) {
				t.Fatalf("err = %v, attendu %q", err, tc.want)
			}
		})
	}
}

func TestUnknownVariable(t *testing.T) {
	_, warns, err := load(t, map[string]string{"SVC_DATABASE_URL": dsn, "SVC_WORKRES": "8", "PATH": "/bin"}, nil)
	if err != nil {
		t.Fatal(err)
	}
	if len(warns) != 1 || !strings.Contains(warns[0], "SVC_WORKRES") {
		t.Fatalf("avertissements : %q", warns)
	}
}

func TestSecretNeverShown(t *testing.T) {
	c, _, err := load(t, map[string]string{"SVC_DATABASE_URL": dsn, "SVC_API_TOKEN_FILE": "/t"}, map[string]string{"/t": "jeton-123\n"})
	if err != nil {
		t.Fatal(err)
	}
	if c.APIToken.Reveal() != "jeton-123" {
		t.Fatalf("Reveal = %q", c.APIToken.Reveal())
	}
	j, _ := json.Marshal(c)
	var logs bytes.Buffer
	slog.New(slog.NewJSONHandler(&logs, nil)).Info("config", "token", c.APIToken, "config", c)
	for _, out := range []string{fmt.Sprintf("%v", c), fmt.Sprintf("%+v", c), fmt.Sprintf("%#v", c),
		fmt.Sprintf("%s|%q|%x", c.APIToken, c.APIToken, c.APIToken), string(j), logs.String()} {
		if strings.Contains(out, "s3cr3t") || strings.Contains(out, "jeton-123") {
			t.Errorf("secret visible : %s", out)
		}
	}
	t.Logf("%%#v : %#v", c.APIToken)
	t.Logf("JSON : %s", j)
}

// Défi : lectures concurrentes pendant les rechargements, sous -race.
func TestHolderConcurrentReload(t *testing.T) {
	levels := []slog.Level{slog.LevelDebug, slog.LevelInfo, slog.LevelWarn}
	var mu sync.Mutex
	i := 0
	h := NewHolder(defaults(), func() (Config, error) {
		mu.Lock()
		defer mu.Unlock()
		c := defaults()
		c.LogLevel = levels[i%len(levels)]
		i++
		return c, nil
	}, slog.New(slog.DiscardHandler))
	var wg sync.WaitGroup
	for range 8 {
		wg.Go(func() {
			for range 1000 {
				if c := h.Get(); c.Workers != 4 {
					t.Error("configuration incohérente")
				}
				_ = h.Level.Level()
			}
		})
	}
	for range 100 {
		h.Reload()
	}
	wg.Wait()
}
```

```go
// Fichier : cmd/svc/main.go
// Commande svc : configuration chargée au démarrage, niveau de journal rechargé sur SIGHUP.
package main

import (
	"errors"
	"flag"
	"fmt"
	"io"
	"log/slog"
	"os"
	"os/signal"
	"syscall"

	"example.com/svc/internal/config"
)

const prefix = "SVC_"

func main() {
	pre := slog.New(slog.NewTextHandler(os.Stderr, nil))
	warn := func(m string) { pre.Warn(m) }
	load := func(args []string, out io.Writer) (config.Config, error) {
		return config.Load(prefix, args, config.OS, out, warn)
	}
	cfg, err := load(os.Args[1:], os.Stderr)
	if errors.Is(err, flag.ErrHelp) {
		os.Exit(0)
	}
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(2)
	}
	reload := func() (config.Config, error) { return load(os.Args[1:], io.Discard) }
	h := config.NewHolder(cfg, reload, pre)
	log := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: &h.Level}))
	log.Info("démarrage", "config", fmt.Sprintf("%+v", *h.Get()))

	hup := make(chan os.Signal, 1)
	signal.Notify(hup, syscall.SIGHUP)
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
	for {
		select {
		case <-hup:
			h.Reload()
			log.Debug("message de débogage") // visible seulement si le niveau est DEBUG
			log.Info("configuration courante", "log_level", h.Get().LogLevel)
		case <-stop:
			log.Info("arrêt")
			return
		}
	}
}
```

```
$ go test -race -v ./internal/config/
--- PASS: TestPrecedence (0.00s)
    --- PASS: TestPrecedence/défauts (0.00s)
    --- PASS: TestPrecedence/fichier (0.00s)
    --- PASS: TestPrecedence/environnement_>_fichier (0.00s)
    --- PASS: TestPrecedence/option_>_environnement (0.00s)
--- PASS: TestErrors (0.00s)
    --- PASS: TestErrors/timeout_hors_plage (0.00s)
    --- PASS: TestErrors/timeout_invalide (0.00s)
    --- PASS: TestErrors/secret_en_double (0.00s)
    --- PASS: TestErrors/champ_inconnu (0.00s)
    --- PASS: TestErrors/secret_dans_le_fichier (0.00s)
--- PASS: TestUnknownVariable (0.00s)
    config_test.go:120: %#v : [masqué]
    config_test.go:121: JSON : {"Addr":":8080","AdminAddr":"127.0.0.1:6060","DatabaseURL":"[masqué]","APIToken":"[masqué]","ShutdownGrace":10000000000,"RequestTimeout":5000000000,"LogLevel":"INFO","Workers":4,"Features":null}
--- PASS: TestSecretNeverShown (0.00s)
--- PASS: TestHolderConcurrentReload (0.00s)
ok  	example.com/svc/internal/config	1.015s
```

## E1 — Un réglage de plus

Dans la version du cours, ajouter `REQUEST_TIMEOUT` touche **quatre** endroits :
- un champ ;
- une valeur par défaut ;
- un bloc de lecture de l'environnement ;
- un `fs.DurationVar` ;

plus une règle de validation. Le nom de la variable et celui de l'option sont écrits à deux endroits
différents, qui peuvent diverger.

La version corrigée décrit chaque réglage **une fois**, dans `settings()` : variable, option, aide et
valeur. Le type générique `value[T]` implémente `flag.Value` (`Set` et `String`). La même valeur sert donc
à lire l'environnement (`s.v.Set(getenv(…))`) et à déclarer l'option (`fs.Var(s.v, …)`).

Ajouter un réglage, c'est maintenant :
- un champ ;
- une valeur par défaut ;
- **une** ligne dans `settings()` ;
- une règle dans `validate`.

Le prix : l'aide affiche `value` au lieu du type (`-workers value` au lieu de `-workers int`). Les
valeurs par défaut affichées sont celles **déjà chargées**, fichier compris :

```
$ SVC_CONFIG_FILE=svc.json ./svc -h        # svc.json : {"log_level": "WARN", "workers": 12}
Usage of svc:
  -addr value
    	adresse d'écoute (SVC_ADDR) (default :8080)
  …
  -log-level value
    	DEBUG, INFO, WARN ou ERROR (SVC_LOG_LEVEL) (default WARN)
  -request-timeout value
    	délai maximal d'une requête (SVC_REQUEST_TIMEOUT) (default 5s)
  …
  -workers value
    	nombre de workers (SVC_WORKERS) (default 12)
```

L'aide reflète ainsi la configuration réelle avant les options. On peut le voir comme une qualité.

## E2 — Un préfixe

Un nom comme `ADDR`, `PORT`, `DEBUG` ou `DATABASE_URL` peut être défini par **autre chose** que le
déploiement du service :
- le shell du développeur ;
- un autre outil ;
- l'orchestrateur. Kubernetes, par exemple, injecte dans chaque conteneur des variables
  `<SERVICE>_PORT`, `<SERVICE>_SERVICE_HOST` pour les services de l'espace de noms.

C'est arrivé en écrivant le cours. Le premier essai de « démarrage sans configuration » a **réussi** : le
shell avait encore une `DATABASE_URL` exportée pour les leçons du niveau 8
(`postgres://dojo:…@127.0.0.1:55439/dojo`), et le service l'a prise. Avec le préfixe `SVC_`, cette variable
n'a plus aucun effet.

Le préfixe est un paramètre de `Load`. Les tests utilisent `SVC_` comme le `main`, et un autre service
réutilisant le paquet choisirait le sien.

## E3 — La faute de frappe

`Load` parcourt `Environ()` : toute variable qui commence par le préfixe et n'est pas dans la liste des noms
connus produit un avertissement.

```
level=WARN msg="variable inconnue ignorée : SVC_WORKRES"
```

Un **avertissement** plutôt qu'une erreur. Une variable inconnue peut être légitime : une variable ajoutée
pour la version **suivante** du service, déployée en avance, ou une variable injectée par l'orchestrateur
avec le même préfixe (voir E2). En faire une erreur rendrait le déploiement de la configuration et celui du
code interdépendants.

En revanche, dans le **fichier**, un champ inconnu est une erreur (`DisallowUnknownFields`). Le fichier
appartient au service et se valide avec lui.

## E4 — Des secrets en fichier, partout

`secret(env, name)` lit `NAME` ou `NAME_FILE`, et refuse les deux à la fois. Le type `Secret` ne s'affiche
**jamais**. Le test `TestSecretNeverShown` vérifie :

| Sortie | Affiche |
|---|---|
| `%v`, `%+v` de la `Config` | `APIToken:[masqué]` |
| `%#v`, `%s`, `%q`, `%x` du secret | `[masqué]` |
| `json.Marshal(cfg)` | `"APIToken":"[masqué]"` |
| `slog` (texte ou JSON) | `[masqué]` |

Chaque méthode couvre une voie de sortie :

- **`Format`** est appelée par `fmt` pour **tous** les verbes : `%v`, `%#v`, `%q`, `%x`… Une simple méthode
  `String` couvre `%v`, `%s`, `%q` et `%x`, mais pas `%#v`, qui affiche la valeur Go brute (mesuré : `"x1"`).
- **`MarshalText`** est utilisée par `encoding/json` (et par `slog.TextHandler`).
- **`LogValue`** est utilisée par `slog` (leçon 6).
- **`Reveal`** est le **seul** accès à la valeur. On la retrouve d'un `grep`, et elle n'apparaît qu'au moment
  d'ouvrir la connexion.

## E5 — Ce que voit `docker inspect`

```
$ docker run -d --name e5a -e DATABASE_URL=postgres://app:s3cr3t@db/app busybox sleep 60
$ docker run -d --name e5b -v ./dsn:/run/secrets/dsn:ro -e DATABASE_URL_FILE=/run/secrets/dsn busybox sleep 60
$ docker inspect --format '{{.Config.Env}}' e5a
[DATABASE_URL=postgres://app:s3cr3t@db/app PATH=…]
$ docker inspect --format '{{.Config.Env}}' e5b
[DATABASE_URL_FILE=/run/secrets/dsn PATH=…]
$ docker run --rm --pid container:e5a busybox sh -c 'tr "\0" "\n" < /proc/1/environ | grep DATABASE'
DATABASE_URL=postgres://app:s3cr3t@db/app
```

Avec `-e`, le secret est dans la **configuration du conteneur**. Le voient :
- quiconque peut lancer `docker inspect`, c'est-à-dire tout membre du groupe `docker` ;
- les outils de supervision qui collectent cette configuration ;
- tout processus qui partage l'espace de processus du conteneur, via `/proc/1/environ`.

Avec le fichier, l'inspection ne montre que le **chemin**. Le fichier lui-même reste lisible dans le
conteneur, et ses droits comptent : ici `-rw-r--r--`, lisible par tous les utilisateurs du conteneur. Un
secret Compose ou Kubernetes se monte avec des droits plus stricts, par exemple 0400.

---

## Exercice intermédiaire — Un fichier de configuration

Dans `config.go` : `loadFile`, `fileConfig` et `Duration`.

1. **La précédence.** Le fichier est lu **avant** l'environnement. Chaque niveau ne remplace que ce qu'il
   fournit. `TestPrecedence` vérifie les quatre niveaux : `workers` vaut 4, puis 6 (fichier), 16
   (environnement), 2 (option). `request_timeout` vaut 3 s depuis le fichier, et 1 s par l'option.
2. **Les champs inconnus.** `DisallowUnknownFields` : `{"workres": 3}` donne `json: unknown field "workres"`.
3. **Les durées.** `time.Duration` est un `int64` : en JSON, `"10s"` serait refusé, et `10` vaudrait 10
   nanosecondes. Le type `Duration` implémente `encoding.TextUnmarshaler`, que `encoding/json` utilise pour
   une chaîne JSON. `slog.Level` l'implémente déjà, d'où `"log_level": "DEBUG"`.
4. **Pas de secret dans le fichier.** Les champs `database_url` et `api_token` sont **déclarés** pour être
   reconnus, puis refusés avec un message clair. Avec `DisallowUnknownFields` seul, l'erreur serait
   `unknown field`, ce qui inciterait à « corriger » en ajoutant le champ. Un fichier de configuration se
   valide dans le dépôt, se copie, s'affiche en revue : ce n'est pas un endroit pour un mot de passe.
5. **Les pointeurs dans `fileConfig`** distinguent « absent » (`nil`, on garde la valeur précédente) de
   « présent à zéro » (`"workers": 0`, qui sera refusé par la validation).

---

## Défi — Changer le niveau de journal sans redémarrer

Le fichier `holder.go` et le `main` ci-dessus. Une exécution réelle :

1. démarrage avec `svc.json = {"log_level": "INFO", …}` ;
2. le fichier passe à `DEBUG`, puis `kill -HUP` ;
3. le fichier devient invalide (`"bavard"`), puis `kill -HUP` ;
4. le fichier passe à `WARN` avec `"workers": 12`, puis `kill -HUP`.

```
level=WARN msg="variable inconnue ignorée : SVC_WORKRES"
level=INFO msg=démarrage config="{Addr::8080 AdminAddr:127.0.0.1:6060 DatabaseURL:[masqué] APIToken:[masqué] ShutdownGrace:10s RequestTimeout:2s LogLevel:INFO Workers:6 Features:[export]}"
level=INFO msg="niveau de journal modifié" avant=INFO après=DEBUG
level=DEBUG msg="message de débogage"
level=INFO msg="configuration courante" log_level=DEBUG
level=ERROR msg="rechargement refusé, configuration inchangée" err="configuration invalide :\nSVC_CONFIG_FILE : slog: level string \"bavard\": unknown name"
level=DEBUG msg="message de débogage"
level=INFO msg="configuration courante" log_level=DEBUG
level=WARN msg="réglages modifiés qui demandent un redémarrage : ignorés" addr=:8080 admin_addr=127.0.0.1:6060 workers=12
level=INFO msg="niveau de journal modifié" avant=DEBUG après=WARN
```

(L'avertissement `SVC_WORKRES` réapparaît à chaque rechargement, il a été retiré ici des lignes suivantes.)

- **Le rechargement relit le fichier, pas l'environnement.** Relire l'environnement n'apporterait rien :
  celui d'un processus en marche ne change pas. Seuls les **fichiers** (configuration, secrets montés que
  l'orchestrateur met à jour) peuvent changer sous un service qui tourne.
- **Une configuration invalide est refusée en entier**, et l'ancienne reste en place. Le service ne
  s'arrête pas : arrêter un service qui marche parce qu'un fichier a été mal édité serait pire que le
  problème.
- **2. Les réglages non modifiables à chaud** (adresses, nombre de workers) sont signalés et **ignorés**. On
  garde les anciennes valeurs dans la configuration publiée, pour qu'elle décrive ce qui tourne réellement.
  Les appliquer demanderait de rouvrir les sockets ou de redimensionner le pool, en somme un redémarrage
  partiel, plus risqué qu'un vrai redémarrage.
- **3. `atomic.Pointer[Config]`** : `Get` renvoie toujours une configuration **complète**, l'ancienne ou la
  nouvelle, jamais un mélange. `slog.LevelVar` est lu par le gestionnaire de journaux à chaque message, sans
  verrou ni course. `TestHolderConcurrentReload` lance 8 goroutines de lecture pendant 100 rechargements, et
  passe sous `-race`.
- **4. Une route `PUT /admin/log-level`.**
  - Avantage : elle est immédiate, sans accès au fichier ni à la machine, et elle peut revenir
    automatiquement au niveau normal après un délai.
  - Risques : n'importe qui atteignant la route peut changer le niveau. `DEBUG` en production peut
    multiplier le volume de journaux, et donc leur coût, et y faire apparaître des données sensibles.
  - On l'expose sur l'adresse d'**administration** seulement (`127.0.0.1:6060`, N10 L10), jamais sur
    l'adresse publique, et on journalise chaque changement avec son auteur.

---

## Quiz

1. Est de la configuration ce qui change d'un déploiement à l'autre sans changer le code (adresses, base,
   secrets, niveaux, tailles). Le comportement métier, les routes et le schéma n'en sont pas.
2. Défauts < (fichier) < environnement < options : chaque niveau ne remplace que ce qu'il fournit.
3. Pour qu'il n'y ait qu'**une** lecture, validée, à un instant connu. Le reste du programme reçoit une
   struct typée : pas de dépendance cachée, pas de valeur relue à mi-parcours.
4. Pour tester le chargement sans modifier l'environnement du processus (`t.Setenv` interdit `t.Parallel`),
   et pour ne dépendre d'aucune variable de la machine de test.
5. Pour qu'un service mal configuré refuse de démarrer, tout de suite, au lieu d'échouer plus tard sur la
   première requête qui utilise la valeur fautive.
6. Pour corriger tout en une fois : chaque cycle « déployer, échouer, corriger » coûte des minutes.
7. Jamais dans une option (visible par `ps`), ni dans un journal, ni dans une erreur, ni dans l'image, ni dans
   un fichier de configuration validé dans le dépôt. On le met dans un fichier monté par l'orchestrateur
   (`/run/secrets/…`), ou à défaut dans une variable d'environnement.
8. `NOM_FILE=/chemin` indique le fichier qui contient la valeur de `NOM`. Le service accepte l'une ou l'autre
   forme, et refuse les deux à la fois.
9. En recopiant la valeur dans son message : `url.Parse("postgres://app:s3cr%t@db/app")` donne
   `parse "postgres://app:s3cr%t@db/app": invalid URL escape "%t"`, mot de passe compris.
10. On construit une configuration complète et validée, puis on la publie d'un coup par un
    `atomic.Pointer[Config]` (ou un `slog.LevelVar` pour le niveau de journal). Les lecteurs voient l'ancienne
    ou la nouvelle, jamais un mélange.
