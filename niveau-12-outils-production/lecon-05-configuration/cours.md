# Leçon 5 — Configuration

> Le même binaire, la même image (leçons 2 et 3), tourne sur le poste du développeur, en recette et en
> production. Ce qui change d'un endroit à l'autre est la **configuration** : adresses, base de données,
> niveaux de journalisation, secrets. Cette leçon montre comment un service Go la lit, une seule fois et à un
> seul endroit, avec une précédence claire. Il la **valide entièrement** avant de démarrer, et il ne laisse
> jamais un secret dans un journal ou une erreur.

## Objectifs

- Distinguer ce qui relève de la configuration et ce qui relève du code.
- Lire la configuration dans les variables d'environnement et les options, avec une précédence explicite.
- La charger dans une struct typée, passée au reste du programme, sans `os.Getenv` dispersés.
- La valider au démarrage et rapporter **toutes** les erreurs d'un coup.
- Lire les secrets dans des fichiers, et ne jamais les afficher.
- Rendre le chargement testable sans toucher à l'environnement du processus.

## Explication

### Qu'est-ce qui est de la configuration ?

Est de la configuration ce qui **change d'un déploiement à l'autre** sans changer le code : adresses d'écoute,
URL de la base, identifiants, niveau de journalisation, nombre de workers, délais. Ne l'est **pas** ce qui
définit le comportement du programme : les règles métier, les routes, le schéma de la base. Les rendre
configurables multiplie les combinaisons à tester sans rien apporter.

Le principe le plus répandu pour les services : la configuration vient de l'**environnement** (variables
d'environnement, fichiers montés par l'orchestrateur), pas de fichiers intégrés à l'image. Une image se
construit une fois (leçon 3) et se déploie partout. Docker, Compose, Kubernetes et les plateformes
d'hébergement savent tous fournir des variables d'environnement.

### Les sources et leur précédence

De la plus faible à la plus forte :

1. **les valeurs par défaut**, dans le code : un service doit démarrer en local avec le minimum de réglages ;
2. **les variables d'environnement** : la configuration du déploiement ;
3. **les options de la ligne de commande** : une exception ponctuelle (`-log-level debug` pour un diagnostic).

Chaque niveau remplace le précédent, et **seulement** pour les valeurs qu'il fournit. On peut ajouter un
fichier de configuration (YAML, TOML…) entre 1 et 2 si la configuration devient grande et structurée. Pour
la plupart des services, ce n'est pas nécessaire. Ce qui compte est que la précédence soit **écrite** et
**testée**.

Les variables d'environnement sont des chaînes. La conversion vers le bon type (durée, entier, niveau de
journal, URL) se fait au chargement, avec les fonctions de la bibliothèque standard :
`time.ParseDuration`, `strconv.Atoi`, `slog.Level.UnmarshalText`, `url.Parse`. Les options utilisent le paquet
`flag` déjà vu, avec un `FlagSet` (leçon 1). `fs.TextVar` accepte toute valeur qui sait se lire depuis du texte,
comme `slog.Level`.

### Une struct, chargée une fois

```go
cfg, err := config.Load(os.Args[1:], config.OS, os.Stderr)
…
run(ctx, cfg)
```

- **Une seule fonction lit l'environnement** : `Load`. Le reste du programme reçoit une `Config` typée, par
  paramètre (N10 L3). Un `os.Getenv("TIMEOUT")` au fond d'un paquet est une dépendance cachée. Personne ne le
  trouve en lisant `main`, et il se lit à chaque appel, peut-être après que l'environnement a changé dans un
  test.
- **L'environnement est injecté.** `Load` reçoit un `Env` (deux fonctions, `Getenv` et `ReadFile`). En
  production, c'est `config.OS`. Dans un test, une map. Pas besoin de `t.Setenv`, qui interdit
  `t.Parallel` (N5).
- **La configuration effective est journalisée au démarrage** (sans secrets). C'est la première chose qu'on
  regarde quand un service se comporte bizarrement : « avec quelle configuration tourne-t-il ? ».

### Valider au démarrage

Un service mal configuré doit **refuser de démarrer**, immédiatement et clairement. Il ne doit pas démarrer
puis échouer à la première requête qui utilise la valeur fautive, peut-être des heures plus tard. On
vérifie donc dans `Load` :

- **les valeurs obligatoires** (`DATABASE_URL`) ;
- **les formats** (durée, entier, URL et son schéma) ;
- **les plages** (`WORKERS` dans [1, 256], un délai d'arrêt raisonnable) ;
- **la cohérence entre valeurs** (l'adresse d'administration différente de l'adresse publique).

Toutes les erreurs sont **rassemblées** (`errors.Join`, N2 L4) et rapportées ensemble. Corriger une erreur,
relancer, découvrir la suivante, et ainsi de suite, est une perte de temps pendant un déploiement. Le
programme sort avec un code **distinct** (2), qu'un script ou un orchestrateur distingue d'un plantage.

### Les secrets

Un secret (mot de passe, jeton, clé) n'est pas une configuration comme les autres.

- **Pas dans une option.** La ligne de commande est visible par tous les utilisateurs de la machine (`ps`,
  `/proc/<pid>/cmdline`).
- **Une variable d'environnement est tolérable, mais exposée.** Elle est héritée par les processus enfants,
  visible dans `docker inspect` et dans `/proc/<pid>/environ` (pour le même utilisateur ou `root`), et
  parfois recopiée dans un rapport de plantage.
- **Un fichier monté est préférable** : un secret Docker ou Compose dans `/run/secrets/…` (leçon 3), un secret
  Kubernetes monté en volume. La convention `NOM_FILE` (déjà utilisée par l'image `postgres` :
  `POSTGRES_PASSWORD_FILE`) indique le chemin du fichier qui contient la valeur. Le service accepte l'une ou
  l'autre forme, et refuse les deux à la fois : laquelle gagnerait ?
- **Jamais dans un journal.** `url.URL.Redacted()` remplace le mot de passe par `xxxxx`. Attention aussi aux
  **erreurs** : `url.Parse` recopie l'URL entière, mot de passe compris, dans son message d'erreur (mesuré
  dans l'exemple). D'où une erreur réécrite (`adresse invalide`) plutôt qu'un `%w`.

La leçon 6 ajoutera le masquage automatique dans `slog`, et la leçon 8 la gestion des secrets.

### Les drapeaux de fonctionnalités

Une fonctionnalité fusionnée mais pas encore ouverte à tous (leçon 1, développement sur le tronc) se
protège par un **drapeau**. Ici, c'est une simple liste : `FEATURES=export,beta`. Le code teste
`cfg.Features["export"]`. Un drapeau se **retire** du code une fois la fonctionnalité ouverte à tous, sinon
les combinaisons s'accumulent. Les drapeaux par utilisateur ou par pourcentage relèvent d'un service dédié,
hors du périmètre de cette leçon.

### Changer la configuration à chaud ?

Le plus simple, et le plus sûr, est de **redémarrer** le service : l'orchestrateur le fait sans interruption
(leçon 9), et la validation au démarrage s'applique. Si un réglage doit vraiment changer sans redémarrage, par
exemple un niveau de journal pendant un incident, on publie une nouvelle `Config` complète et validée par
un `atomic.Pointer[Config]` (N6 L7). Les lecteurs ne voient jamais une configuration à moitié modifiée.

## Exemple

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27
```

```go
// Fichier : internal/config/config.go
// Package config lit, valide et décrit la configuration du service.
//
// Précédence, de la plus faible à la plus forte : valeurs par défaut < variables
// d'environnement < options de la ligne de commande.
package config

import (
	"errors"
	"flag"
	"fmt"
	"io"
	"log/slog"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

type Config struct {
	Addr          string        // adresse d'écoute publique
	AdminAddr     string        // adresse d'administration (métriques, profils)
	DatabaseURL   *url.URL      // contient le mot de passe : jamais affichée en clair
	ShutdownGrace time.Duration // délai d'arrêt propre (leçon 3)
	LogLevel      slog.Level
	Workers       int
	Features      map[string]bool // drapeaux de fonctionnalités : FEATURES=export,beta
}

// Env donne accès aux variables d'environnement et aux fichiers : injectable dans les tests.
type Env struct {
	Getenv   func(string) string
	ReadFile func(string) ([]byte, error)
}

// OS est l'environnement réel du processus.
var OS = Env{Getenv: os.Getenv, ReadFile: os.ReadFile}

func defaults() Config {
	return Config{
		Addr:          ":8080",
		AdminAddr:     "127.0.0.1:6060",
		ShutdownGrace: 10 * time.Second,
		LogLevel:      slog.LevelInfo,
		Workers:       4,
		Features:      map[string]bool{},
	}
}

// Load construit la configuration et la valide. Toutes les erreurs sont rapportées ensemble.
func Load(args []string, env Env, out io.Writer) (Config, error) {
	c := defaults()
	var errs []error
	bad := func(name string, err error) { errs = append(errs, fmt.Errorf("%s : %w", name, err)) }

	// 1. Variables d'environnement.
	if v := env.Getenv("ADDR"); v != "" {
		c.Addr = v
	}
	if v := env.Getenv("ADMIN_ADDR"); v != "" {
		c.AdminAddr = v
	}
	if v := env.Getenv("SHUTDOWN_GRACE"); v != "" {
		if d, err := time.ParseDuration(v); err != nil {
			bad("SHUTDOWN_GRACE", err)
		} else {
			c.ShutdownGrace = d
		}
	}
	if v := env.Getenv("LOG_LEVEL"); v != "" {
		if err := c.LogLevel.UnmarshalText([]byte(v)); err != nil {
			bad("LOG_LEVEL", err)
		}
	}
	if v := env.Getenv("WORKERS"); v != "" {
		if n, err := strconv.Atoi(v); err != nil {
			bad("WORKERS", err)
		} else {
			c.Workers = n
		}
	}
	for f := range strings.SplitSeq(env.Getenv("FEATURES"), ",") {
		if f = strings.TrimSpace(f); f != "" {
			c.Features[f] = true
		}
	}
	// Un secret se lit de préférence dans un fichier (DATABASE_URL_FILE) : il n'apparaît
	// ni dans l'environnement du processus, ni dans `docker inspect`.
	dsn := env.Getenv("DATABASE_URL")
	if p := env.Getenv("DATABASE_URL_FILE"); p != "" {
		if dsn != "" {
			bad("DATABASE_URL_FILE", errors.New("DATABASE_URL est aussi définie : en choisir une"))
		}
		b, err := env.ReadFile(p)
		if err != nil {
			bad("DATABASE_URL_FILE", err)
		}
		dsn = strings.TrimSpace(string(b))
	}

	// 2. Options de la ligne de commande : elles l'emportent.
	fs := flag.NewFlagSet("svc", flag.ContinueOnError)
	fs.SetOutput(out)
	fs.StringVar(&c.Addr, "addr", c.Addr, "adresse d'écoute (ADDR)")
	fs.StringVar(&c.AdminAddr, "admin-addr", c.AdminAddr, "adresse d'administration (ADMIN_ADDR)")
	fs.DurationVar(&c.ShutdownGrace, "shutdown-grace", c.ShutdownGrace, "délai d'arrêt propre (SHUTDOWN_GRACE)")
	fs.TextVar(&c.LogLevel, "log-level", c.LogLevel, "DEBUG, INFO, WARN ou ERROR (LOG_LEVEL)")
	fs.IntVar(&c.Workers, "workers", c.Workers, "nombre de workers (WORKERS)")
	if err := fs.Parse(args); err != nil {
		return Config{}, err // -h compris : flag.ErrHelp
	}
	if fs.NArg() > 0 {
		bad("arguments", fmt.Errorf("inattendus : %q", fs.Args()))
	}

	// 3. Validation.
	if dsn == "" {
		bad("DATABASE_URL", errors.New("obligatoire (ou DATABASE_URL_FILE)"))
	} else if u, err := url.Parse(dsn); err != nil {
		bad("DATABASE_URL", errors.New("adresse invalide")) // pas %w : l'erreur de url.Parse recopie le mot de passe
	} else if u.Scheme != "postgres" && u.Scheme != "postgresql" {
		bad("DATABASE_URL", fmt.Errorf("schéma %q, attendu postgres", u.Scheme))
	} else {
		c.DatabaseURL = u
	}
	if c.Workers < 1 || c.Workers > 256 {
		bad("WORKERS", fmt.Errorf("%d hors de [1, 256]", c.Workers))
	}
	if c.ShutdownGrace <= 0 || c.ShutdownGrace > time.Minute {
		bad("SHUTDOWN_GRACE", fmt.Errorf("%v hors de ]0, 1m]", c.ShutdownGrace))
	}
	if c.Addr == c.AdminAddr {
		bad("ADMIN_ADDR", errors.New("doit différer de ADDR"))
	}
	if len(errs) > 0 {
		return Config{}, fmt.Errorf("configuration invalide :\n%w", errors.Join(errs...))
	}
	return c, nil
}

// String décrit la configuration effective, sans secret : c'est ce qu'on journalise au démarrage.
func (c Config) String() string {
	db := "(aucune)"
	if c.DatabaseURL != nil {
		db = c.DatabaseURL.Redacted() // postgres://user:xxxxx@hôte/base
	}
	return fmt.Sprintf("addr=%s admin=%s db=%s grace=%v log=%v workers=%d features=%v",
		c.Addr, c.AdminAddr, db, c.ShutdownGrace, c.LogLevel, c.Workers, c.Features)
}
```

```go
// Fichier : internal/config/config_test.go
package config

import (
	"errors"
	"flag"
	"io"
	"io/fs"
	"strings"
	"testing"
	"time"
)

// fakeEnv construit un environnement à partir d'une map, et des fichiers à partir d'une autre.
func fakeEnv(vars, files map[string]string) Env {
	return Env{
		Getenv: func(k string) string { return vars[k] },
		ReadFile: func(p string) ([]byte, error) {
			if s, ok := files[p]; ok {
				return []byte(s), nil
			}
			return nil, fs.ErrNotExist
		},
	}
}

const dsn = "postgres://app:s3cr3t@db:5432/app"

func TestPrecedence(t *testing.T) {
	for _, tc := range []struct {
		name    string
		vars    map[string]string
		args    []string
		workers int
	}{
		{"défaut", map[string]string{"DATABASE_URL": dsn}, nil, 4},
		{"environnement", map[string]string{"DATABASE_URL": dsn, "WORKERS": "16"}, nil, 16},
		{"option > environnement", map[string]string{"DATABASE_URL": dsn, "WORKERS": "16"}, []string{"-workers", "2"}, 2},
	} {
		t.Run(tc.name, func(t *testing.T) {
			c, err := Load(tc.args, fakeEnv(tc.vars, nil), io.Discard)
			if err != nil {
				t.Fatal(err)
			}
			if c.Workers != tc.workers {
				t.Fatalf("Workers = %d, attendu %d", c.Workers, tc.workers)
			}
		})
	}
}

func TestSecretFile(t *testing.T) {
	env := fakeEnv(map[string]string{"DATABASE_URL_FILE": "/run/secrets/dsn"},
		map[string]string{"/run/secrets/dsn": dsn + "\n"})
	c, err := Load(nil, env, io.Discard)
	if err != nil {
		t.Fatal(err)
	}
	if c.DatabaseURL.String() != dsn {
		t.Fatalf("DatabaseURL = %s", c.DatabaseURL)
	}
}

func TestValidationReportsEverything(t *testing.T) {
	env := fakeEnv(map[string]string{"WORKERS": "0", "SHUTDOWN_GRACE": "abc", "LOG_LEVEL": "bavard"}, nil)
	_, err := Load(nil, env, io.Discard)
	if err == nil {
		t.Fatal("configuration invalide acceptée")
	}
	for _, want := range []string{"DATABASE_URL", "WORKERS", "SHUTDOWN_GRACE", "LOG_LEVEL"} {
		if !strings.Contains(err.Error(), want) {
			t.Errorf("erreur sans %s :\n%v", want, err)
		}
	}
}

func TestNoSecretLeak(t *testing.T) {
	// Ni l'affichage de la configuration, ni une erreur de validation ne doivent contenir le mot de passe.
	c, err := Load(nil, fakeEnv(map[string]string{"DATABASE_URL": dsn}, nil), io.Discard)
	if err != nil {
		t.Fatal(err)
	}
	if strings.Contains(c.String(), "s3cr3t") {
		t.Errorf("String() contient le secret : %s", c)
	}
	_, err = Load(nil, fakeEnv(map[string]string{"DATABASE_URL": "postgres://app:s3cr%t@db/app"}, nil), io.Discard)
	if err == nil || strings.Contains(err.Error(), "s3cr") {
		t.Errorf("erreur qui contient le secret : %v", err)
	}
}

func TestHelp(t *testing.T) {
	var out strings.Builder
	_, err := Load([]string{"-h"}, fakeEnv(nil, nil), &out)
	if !errors.Is(err, flag.ErrHelp) || !strings.Contains(out.String(), "-workers") {
		t.Fatalf("err = %v, aide = %q", err, out.String())
	}
}

func TestDefaults(t *testing.T) {
	c, err := Load(nil, fakeEnv(map[string]string{"DATABASE_URL": dsn}, nil), io.Discard)
	if err != nil {
		t.Fatal(err)
	}
	if c.ShutdownGrace != 10*time.Second || c.Addr != ":8080" {
		t.Fatalf("%v", c)
	}
}
```

```go
// Fichier : cmd/svc/main.go
// Commande svc : charge sa configuration, l'affiche, et s'arrête au moindre problème.
package main

import (
	"errors"
	"flag"
	"fmt"
	"os"

	"example.com/svc/internal/config"
)

func main() {
	cfg, err := config.Load(os.Args[1:], config.OS, os.Stderr)
	if errors.Is(err, flag.ErrHelp) {
		os.Exit(0)
	}
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(2) // code distinct : « mal configuré », pas « planté »
	}
	fmt.Println("configuration :", cfg)
	// … run(ctx, cfg) : le reste du service ne lit plus jamais l'environnement.
}
```

```
$ go test -v ./internal/config/
--- PASS: TestPrecedence (0.00s)
--- PASS: TestSecretFile (0.00s)
--- PASS: TestValidationReportsEverything (0.00s)
--- PASS: TestNoSecretLeak (0.00s)
--- PASS: TestHelp (0.00s)
--- PASS: TestDefaults (0.00s)
ok  	example.com/svc/internal/config	0.002s
```

Le programme, dans différentes situations :

```
$ ./svc
configuration invalide :
DATABASE_URL : obligatoire (ou DATABASE_URL_FILE)
$ echo $?
2
$ DATABASE_URL='postgres://app:s3cr3t@db:5432/app' FEATURES=export,beta LOG_LEVEL=debug ./svc -workers 8
configuration : addr=:8080 admin=127.0.0.1:6060 db=postgres://app:xxxxx@db:5432/app grace=10s log=DEBUG workers=8 features=map[beta:true export:true]
$ DATABASE_URL='postgres://a:b@db/x' WORKERS=16 ./svc -workers 2
configuration : addr=:8080 admin=127.0.0.1:6060 db=postgres://a:xxxxx@db/x grace=10s log=INFO workers=2 features=map[]
$ DATABASE_URL='mysql://x' WORKERS=abc SHUTDOWN_GRACE=2h LOG_LEVEL=verbeux ADDR=127.0.0.1:6060 ./svc extra
configuration invalide :
LOG_LEVEL : slog: level string "verbeux": unknown name
WORKERS : strconv.Atoi: parsing "abc": invalid syntax
arguments : inattendus : ["extra"]
DATABASE_URL : schéma "mysql", attendu postgres
SHUTDOWN_GRACE : 2h0m0s hors de ]0, 1m]
ADMIN_ADDR : doit différer de ADDR
$ DATABASE_URL_FILE=/run/secrets/dsn ./svc
configuration : addr=:8080 admin=127.0.0.1:6060 db=postgres://app:xxxxx@db/app grace=10s log=INFO workers=4 features=map[]
$ ./svc -h
Usage of svc:
  -addr string
    	adresse d'écoute (ADDR) (default ":8080")
  -admin-addr string
    	adresse d'administration (ADMIN_ADDR) (default "127.0.0.1:6060")
  -log-level value
    	DEBUG, INFO, WARN ou ERROR (LOG_LEVEL) (default INFO)
  -shutdown-grace duration
    	délai d'arrêt propre (SHUTDOWN_GRACE) (default 10s)
  -workers int
    	nombre de workers (WORKERS) (default 4)
```

(Pour l'avant-dernier exemple, le fichier contient `postgres://app:depuis-fichier@db/app`.)

Et l'erreur de `url.Parse` que `Load` ne recopie pas :

```go
_, err := url.Parse("postgres://app:s3cr%t@db/app")
fmt.Println(err)
```

```
parse "postgres://app:s3cr%t@db/app": invalid URL escape "%t"
```

Le mot de passe (`s3cr%t`) figure en clair dans le message. Journalisé tel quel, il serait copié dans le
système de journaux, consultable par bien plus de monde que ceux qui ont accès au secret.

## Explication du code

| Élément | Rôle |
|---|---|
| `defaults()` | Les valeurs par défaut, en un seul endroit, lisibles. |
| `Env{Getenv, ReadFile}` | Les deux accès au monde extérieur, injectés. `config.OS` utilise `os.Getenv` et `os.ReadFile`. |
| `bad(nom, err)` | Accumule les erreurs au lieu de s'arrêter à la première. |
| `if d, err := …; err != nil { bad } else { c.X = d }` | La valeur n'est remplacée que si elle est valide. Sinon, une valeur à zéro provoquerait une seconde erreur (« 0 hors de [1, 256] ») qui masquerait la vraie. |
| `fs.StringVar(&c.Addr, "addr", c.Addr, …)` | La valeur par défaut de l'option est la valeur **déjà chargée** (défaut ou environnement) : l'option ne la remplace que si elle est donnée. C'est ce qui réalise la précédence. |
| `flag.ContinueOnError` + `fs.SetOutput(out)` | `Load` renvoie une erreur (dont `flag.ErrHelp` pour `-h`) au lieu d'appeler `os.Exit`, et écrit l'aide où on lui dit : testable. |
| `DATABASE_URL_FILE` | Le secret lu dans un fichier ; `TrimSpace` retire le saut de ligne final que la plupart des éditeurs ajoutent. |
| `errors.New("adresse invalide")` sans `%w` | Ne pas propager un message d'erreur qui contient le secret. |
| `errors.Join(errs...)` | Une seule erreur qui contient toutes les autres, une par ligne. |
| `os.Exit(2)` | « Configuration invalide », distinct d'un plantage. |
| `Config.String()` et `Redacted()` | La description journalisée, sans le mot de passe. |

## Erreurs fréquentes

1. **Des `os.Getenv` dans tout le code**, lus au moment de l'usage, sans valeur par défaut ni validation.
2. **Une valeur invalide ignorée en silence** (`strconv.Atoi` en erreur, on garde 0) : le service tourne avec
   0 worker.
3. **S'arrêter à la première erreur de configuration** : cinq déploiements pour corriger cinq variables.
4. **Démarrer quand même** avec une configuration incomplète, et échouer à la première requête.
5. **Un secret en option de ligne de commande**, ou affiché au démarrage avec le reste de la configuration.
6. **Propager une erreur qui contient le secret** (`url.Parse`, une erreur de connexion qui recopie la chaîne
   de connexion).
7. **Une configuration différente selon l'environnement dans l'image** (`config-prod.yaml` copié au build) :
   une image par environnement, et ce qu'on teste n'est plus ce qu'on déploie.
8. **`t.Setenv` partout dans les tests**, faute d'injection : des tests qui ne peuvent pas être parallèles.

## Bonnes pratiques Go

- Un paquet `config` : `Load(args, env, out) (Config, error)`, appelé une fois par `main`.
- Précédence écrite et testée : défauts < environnement < options.
- Validation complète au démarrage, toutes les erreurs ensemble, code de sortie distinct.
- Secrets par fichier (`*_FILE`), jamais en option, jamais journalisés, jamais recopiés dans une erreur.
- La configuration effective journalisée au démarrage, secrets masqués.
- Une seule image pour tous les environnements ; seule la configuration change.

## Ce que je dois retenir

- La configuration est ce qui change d'un déploiement à l'autre ; elle vient de l'environnement.
- Elle est lue une fois, dans une struct typée, et passée au reste du programme.
- Un service mal configuré refuse de démarrer, et dit **tout** ce qui ne va pas.
- Les secrets viennent de fichiers montés et ne sortent jamais : ni journal, ni erreur, ni ligne de
  commande.
- Injecter l'environnement rend le chargement testable sans `t.Setenv`.

---

➡️ [Exercices](exercices.md)
