# Leçon 3 — Injection de dépendances

> « Injection de dépendances » évoque, dans d'autres écosystèmes, des conteneurs, des annotations et de
> la magie au démarrage. En Go, c'est beaucoup plus simple : **chaque composant reçoit ce dont il a
> besoin par son constructeur**, et **un seul endroit**, `main`, construit le graphe. Pas de framework,
> pas de réflexion, et le compilateur vérifie tout. Cette leçon montre ce style, ses deux outils (le
> constructeur et les options fonctionnelles), et un `main` qui se teste.

## Objectifs

1. Passer toutes les dépendances par les constructeurs, sans état global ni `init` à effets de bord.
2. Faire de `main` la **racine de composition** : le seul endroit qui connaît le graphe entier.
3. Écrire un `run(ctx, args, getenv, stdout…) error` testable, et un `main` de trois lignes.
4. Utiliser les **options fonctionnelles** pour les réglages facultatifs.
5. Fermer les ressources dans le bon ordre.
6. Savoir ce qu'apporte un outil comme `wire`, et pourquoi on s'en passe le plus souvent.

## Explication

### Une dépendance = un paramètre de constructeur

```go
func NewService(s *Store, n *Notifier, now Clock) *Service
```

Le service ne crée **rien** : ni connexion, ni client, ni horloge. Il les reçoit. Conséquences :

- en test, on lui passe des faux (N5 L3) ou une horloge fixe ;
- ses dépendances sont **visibles** dans sa signature : un constructeur à neuf paramètres est un signal
  que le composant en fait trop ;
- aucun ordre d'initialisation caché : le compilateur refuse d'appeler `NewService` sans un `*Store`.

Ce qu'on évite par là :

- les **variables globales** (`var DB *sql.DB`) que n'importe quel paquet modifie et que les tests
  partagent ;
- les **`init()` à effets de bord** (ouvrir une connexion, lire l'environnement, lancer une goroutine) :
  ils s'exécutent à l'import, avant `main`, dans un ordre que personne ne maîtrise, y compris pendant
  `go test` ;
- les **singletons** et « localisateurs de services » (`registry.Get("db")`) : la dépendance existe, mais
  elle est cachée.

### La racine de composition

`main` (ici, `run`) est le **seul** endroit qui :

1. lit la configuration (environnement, arguments) ;
2. ouvre les ressources (base, clients) ;
3. construit chaque composant avec ses dépendances, des feuilles vers la racine ;
4. lance le programme (serveur HTTP, commande) ;
5. ferme tout, dans l'ordre **inverse** de l'ouverture (`defer`, N2 L7).

Le reste du code ne connaît que ses dépendances directes. Quand le graphe change (un cache devant le
stockage, un autre notificateur), seule la racine change.

### `main` testable : la fonction `run`

```go
func main() {
	if err := run(context.Background(), os.Args[1:], os.Getenv, os.Stdout, time.Now); err != nil {
		fmt.Fprintln(os.Stderr, "erreur :", err)
		os.Exit(1)
	}
}
```

`run` reçoit **tout ce que `main` lirait dans le monde extérieur** : arguments, environnement (une
fonction, pas `os.Getenv` en dur), sorties, horloge, contexte (annulé par un signal, N7 L7). `main` ne fait
que brancher le vrai monde. Bénéfices :

- le démarrage complet (configuration invalide, ressource absente, assemblage) se teste comme une
  fonction ordinaire, sans lancer de processus ;
- `os.Exit` n'est appelé qu'à un endroit : les `defer` de `run` s'exécutent toujours (un `os.Exit` ou un
  `log.Fatal` au milieu de `run` les sauterait) ;
- une erreur de démarrage est une **valeur** qu'on peut vérifier, pas un message sur la sortie d'erreur.

### Les options fonctionnelles

Pour un composant qui a des réglages **facultatifs** avec des valeurs par défaut sensées :

```go
type Option func(*Notifier)

func WithPrefix(p string) Option         { return func(n *Notifier) { n.prefix = p } }
func WithTimeout(d time.Duration) Option { return func(n *Notifier) { n.timeout = d } }

func NewNotifier(out io.Writer, opts ...Option) *Notifier

n := NewNotifier(os.Stdout)                                    // défauts
n := NewNotifier(os.Stdout, WithTimeout(500*time.Millisecond)) // un réglage
```

Les dépendances **obligatoires** restent des paramètres positionnels (`out`) ; les réglages facultatifs
sont des options. Ajouter une option ne casse aucun appelant. L'alternative, une struct de configuration
(`NewNotifier(out, NotifierConfig{Timeout: …})`), est tout aussi idiomatique et plus simple quand les
réglages viennent d'un fichier de configuration ; le piège de la struct est de distinguer « non
renseigné » de « zéro voulu » (un `Timeout` de 0 : défaut, ou pas de délai ?).

### Et `wire` ?

`github.com/google/wire` génère (leçon N9 L4) le code d'assemblage à partir de la liste des constructeurs :
on déclare les « fournisseurs », il écrit la fonction qui les appelle dans le bon ordre. Utile quand le
graphe compte des dizaines de composants et change souvent. Pour la plupart des services, la racine de
composition écrite à la main fait quelques dizaines de lignes lisibles, et c'est préférable : on voit
d'un coup d'œil qui dépend de quoi. Les conteneurs d'injection par réflexion, eux, déplacent les erreurs
d'assemblage de la compilation au démarrage, sans rien apporter que le compilateur ne fasse déjà.

### Quand ne pas le faire

- **Un script ou un petit outil** : lire `os.Getenv` dans `main` et passer les valeurs suffit ; une
  architecture de constructeurs pour 50 lignes est de la cérémonie.
- **Des options fonctionnelles pour un seul réglage** stable : un paramètre suffit.
- **Injecter ce qui ne varie jamais** et n'a aucun effet extérieur : une fonction pure (`strings.ToLower`,
  un calcul de pénalité) s'appelle directement. On injecte ce qui touche le monde extérieur (réseau,
  disque, horloge, aléa) ou ce qu'on veut remplacer en test.

## Exemple

Un programme complet : configuration lue depuis une fonction `getenv`, une ressource à fermer, un
composant à options, un service assemblé dans `run`. La fonction d'exemple de test (N5 L5) exécute trois
démarrages et sa sortie est vérifiée par `go test`.

```go
// Fichier : main.go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"os"
	"time"
)

// --- Les composants : chacun reçoit ses dépendances par son constructeur. ---

type Clock func() time.Time

// Notifier envoie des messages ; ses réglages passent par des options fonctionnelles.
type Notifier struct {
	out     io.Writer
	prefix  string
	timeout time.Duration
}

type Option func(*Notifier)

func WithPrefix(p string) Option         { return func(n *Notifier) { n.prefix = p } }
func WithTimeout(d time.Duration) Option { return func(n *Notifier) { n.timeout = d } }

func NewNotifier(out io.Writer, opts ...Option) *Notifier {
	n := &Notifier{out: out, prefix: "[notif]", timeout: 2 * time.Second} // valeurs par défaut sûres
	for _, o := range opts {
		o(n)
	}
	return n
}

func (n *Notifier) Notify(ctx context.Context, to, msg string) error {
	ctx, cancel := context.WithTimeout(ctx, n.timeout)
	defer cancel()
	if err := ctx.Err(); err != nil {
		return err
	}
	_, err := fmt.Fprintf(n.out, "%s %s : %s\n", n.prefix, to, msg)
	return err
}

// Store simule une ressource à fermer (une base, un fichier).
type Store struct {
	out   io.Writer
	loans map[string]int
}

func OpenStore(out io.Writer, dsn string) (*Store, error) {
	if dsn == "" {
		return nil, errors.New("DSN manquant")
	}
	fmt.Fprintln(out, "stockage ouvert :", dsn)
	return &Store{out: out, loans: map[string]int{}}, nil
}

func (s *Store) Close() error { fmt.Fprintln(s.out, "stockage fermé"); return nil }

// Service : le cas d'usage. Il ne crée RIEN lui-même.
type Service struct {
	store  *Store
	notify interface {
		Notify(ctx context.Context, to, msg string) error
	}
	now Clock
}

func NewService(s *Store, n *Notifier, now Clock) *Service {
	return &Service{store: s, notify: n, now: now}
}

func (s *Service) Borrow(ctx context.Context, member, book string) error {
	s.store.loans[member]++
	due := s.now().Add(21 * 24 * time.Hour).Format(time.DateOnly)
	return s.notify.Notify(ctx, member, fmt.Sprintf("« %s » à rendre le %s", book, due))
}

// --- La racine de composition : le SEUL endroit qui connaît tout le graphe. ---

type Config struct {
	DSN          string
	NotifyPrefix string
}

func loadConfig(getenv func(string) string) (Config, error) {
	c := Config{DSN: getenv("DATABASE_URL"), NotifyPrefix: getenv("NOTIFY_PREFIX")}
	if c.DSN == "" {
		return c, errors.New("DATABASE_URL obligatoire")
	}
	return c, nil
}

// run reçoit tout ce que main lit dans le monde extérieur : il est testable tel quel.
func run(ctx context.Context, args []string, getenv func(string) string, out io.Writer, now Clock) error {
	cfg, err := loadConfig(getenv)
	if err != nil {
		return err
	}
	store, err := OpenStore(out, cfg.DSN)
	if err != nil {
		return err
	}
	defer store.Close() // fermé en DERNIER : ouvert en premier

	var opts []Option
	if cfg.NotifyPrefix != "" {
		opts = append(opts, WithPrefix(cfg.NotifyPrefix))
	}
	svc := NewService(store, NewNotifier(out, opts...), now)

	if len(args) != 2 {
		return fmt.Errorf("usage : emprunt <adhérent> <livre>")
	}
	return svc.Borrow(ctx, args[0], args[1])
}

func main() {
	if err := run(context.Background(), os.Args[1:], os.Getenv, os.Stdout, time.Now); err != nil {
		fmt.Fprintln(os.Stderr, "erreur :", err)
		os.Exit(1)
	}
}
```

```go
// Fichier : main_test.go
package main

import (
	"context"
	"fmt"
	"os"
	"time"
)

// Example_run : trois démarrages avec des environnements différents. La sortie est
// VÉRIFIÉE par go test (N5 L5) : run est testable parce qu'il ne lit rien lui-même.
func Example_run() {
	fixed := func() time.Time { return time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC) }
	envs := []map[string]string{
		{"DATABASE_URL": "postgres://local/biblio"},
		{"DATABASE_URL": "postgres://local/biblio", "NOTIFY_PREFIX": "[courriel]"},
		{},
	}
	for i, env := range envs {
		fmt.Printf("--- démarrage %d\n", i+1)
		err := run(context.Background(), []string{"ada", "La Peste"},
			func(k string) string { return env[k] }, os.Stdout, fixed)
		fmt.Println("erreur :", err)
	}
	// Output:
	// --- démarrage 1
	// stockage ouvert : postgres://local/biblio
	// [notif] ada : « La Peste » à rendre le 2026-03-22
	// stockage fermé
	// erreur : <nil>
	// --- démarrage 2
	// stockage ouvert : postgres://local/biblio
	// [courriel] ada : « La Peste » à rendre le 2026-03-22
	// stockage fermé
	// erreur : <nil>
	// --- démarrage 3
	// erreur : DATABASE_URL obligatoire
}
```
`go test -v ./...` :

```
--- PASS: Example_run (0.00s)
PASS
```

Et en vrai :

```
$ DATABASE_URL=postgres://x go run . ada "Vol de nuit"
stockage ouvert : postgres://x
[notif] ada : « Vol de nuit » à rendre le 2026-11-24
stockage fermé
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `NewService(s, n, now)` | Les trois dépendances, visibles ; le service ne crée rien. |
| `Clock` (`func() time.Time`) | L'horloge est une dépendance comme les autres : `time.Now` en production, une date fixe en test. |
| `notify interface{ Notify(…) }` dans `Service` | Le service n'exige qu'une méthode : on peut lui passer un faux (leçon 4). |
| `NewNotifier(out, opts...)` | `out` obligatoire, préfixe et délai facultatifs avec défauts sûrs. |
| `loadConfig(getenv)` | La configuration validée **avant** d'ouvrir quoi que ce soit ; l'absence du DSN arrête tout avec un message clair. |
| `defer store.Close()` juste après l'ouverture | Fermé en dernier, y compris si une étape suivante échoue. |
| `run(ctx, args, getenv, out, now)` | Toutes les entrées du monde extérieur en paramètres. |
| `main` en trois lignes | Branche le vrai monde, et seul endroit qui appelle `os.Exit`. |
| `Example_run` | Trois démarrages complets, sortie vérifiée, sans processus ni variable d'environnement réelle. |

## Erreurs fréquentes

1. **Des variables globales** pour la base, la configuration, le journal.
2. **`init()` qui ouvre une connexion** ou lit l'environnement.
3. **Un composant qui crée ses propres dépendances** (`NewService()` qui fait `sql.Open`).
4. **`log.Fatal` / `os.Exit` ailleurs que dans `main`** : les `defer` sautent.
5. **Lire `os.Getenv` partout** au lieu d'une configuration lue et validée une fois.
6. **Un constructeur à douze paramètres** : le composant fait trop de choses.
7. **Un conteneur d'injection par réflexion** là où vingt lignes de `main` suffisent.
8. **Fermer dans le mauvais ordre** (le stockage avant le serveur qui l'utilise encore).

## Bonnes pratiques Go

- Dépendances obligatoires en paramètres de constructeur ; réglages facultatifs en options ou en struct.
- `main` minimal ; `run` reçoit le monde extérieur et rend une erreur.
- Configuration lue et validée une fois, au début de `run`.
- Ressources fermées par `defer`, dans l'ordre inverse de leur ouverture.
- Injecter ce qui touche le monde extérieur ; appeler directement le reste.
- `wire` seulement quand le graphe est grand et mouvant.

## Ce que je dois retenir

- En Go, l'injection de dépendances = des constructeurs + une racine de composition dans `main`.
- Pas d'état global, pas d'`init` à effets de bord, pas de dépendance cachée.
- `run(ctx, args, getenv, stdout…) error` rend le démarrage testable.
- Options fonctionnelles : réglages facultatifs, défauts sûrs, appelants non cassés par les ajouts.
- Le compilateur vérifie l'assemblage ; un framework n'est presque jamais nécessaire.

---

➡️ [Exercices](exercices.md)
