# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Premier test

```go
package slug_test

import (
	"testing"

	"exemple/slug"
)

func TestSlug(t *testing.T) {
	if got, want := slug.Slug("Bonjour le Monde"), "bonjour-le-monde"; got != want {
		t.Errorf("Slug(%q) = %q, attendu %q", "Bonjour le Monde", got, want)
	}
	if got, want := slug.Slug("Ça, c'est l'été!"), "ca-c-est-l-ete"; got != want {
		t.Errorf("Slug(%q) = %q, attendu %q", "Ça, c'est l'été!", got, want)
	}
	if got := slug.Slug("!!!"); got != "" {
		t.Errorf("Slug(%q) = %q, attendu \"\"", "!!!", got)
	}
}
```

`(cached)` : `go test` a reconnu que ni le code ni les tests n'ont changé depuis la dernière
exécution réussie, et réaffiche le résultat mémorisé sans rien exécuter. `go test -count=1`
force l'exécution. Le cache ne s'applique qu'en mode « liste de paquets » (`go test ./...`,
`go test .`), pas avec `go test` sans argument.

Remarquer la répétition entre les trois vérifications : c'est exactement ce que la leçon 2
supprime.

## E2 — `Error` ou `Fatal`

```go
func TestLoadConfig(t *testing.T) {
	path := filepath.Join(t.TempDir(), "app.conf")
	os.WriteFile(path, []byte("port=8080\ndebug=false\nname=api\n"), 0o644)

	cfg, err := Load(path)
	if err != nil {
		t.Fatalf("Load(%q) : %v", path, err)     // Fatal : cfg est inutilisable
	}
	if cfg.Port != 8080 {
		t.Errorf("Port = %d, attendu 8080", cfg.Port)  // Error : on veut voir les trois
	}
	if cfg.Debug {
		t.Errorf("Debug = %v, attendu false", cfg.Debug)
	}
	if cfg.Name != "api" {
		t.Errorf("Name = %q, attendu %q", cfg.Name, "api")
	}
}
```

(a) Fichier absent : **une** ligne (`Fatal`), puis le test s'arrête — continuer aurait
déréférencé un `cfg` nil, et ajouté une panique au diagnostic. (b) Deux champs faux : **deux**
lignes, une par champ. Avec `Fatal` partout, on n'aurait vu que la première, corrigé, relancé,
découvert la seconde.

## E3 — `t.Helper`

Mesuré :

```
=== RUN   TestA
    p_test.go:4: got 1 want 2     ← sans Helper : la ligne DANS assertEqual
    p_test.go:10: got 1 want 2    ← avec Helper : la ligne du test qui appelle
```

Sans `Helper`, tous les échecs de tous les tests pointent la même ligne 4 : inutile pour savoir
**quel appel** a échoué.

Version générique :

```go
func assertEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got != want {
		t.Errorf("obtenu %v, attendu %v", got, want)
	}
}
```

Ce genre d'utilitaire est acceptable en interne. Mais il appauvrit le message (il ne connaît ni
la fonction ni l'entrée) : c'est l'argument de l'équipe Go contre les bibliothèques d'assertions.

## E4 — Test externe

```go
// export_test.go — compilé SEULEMENT pendant les tests, dans le paquet interne
package stats

var Quantile = quantile // expose la fonction non exportée aux tests externes
```

```go
// stats_test.go
package stats_test

func TestQuantile(t *testing.T) {
	got := stats.Quantile([]float64{1, 2, 3}, 0.5)
	…
}
```

C'est un patron de la bibliothèque standard (`net/http`, `strings` l'utilisent). Les fichiers
`_test.go` n'étant jamais compilés dans le binaire, cette exposition n'existe que pour les tests.

## E5 — Cleanup

Mesuré :

```
=== RUN   TestC
    p_test.go:12: corps
    p_test.go:11: nettoyage 2
    p_test.go:11: nettoyage 1
--- PASS: TestC (0.00s)
```

Ordre **LIFO**, comme `defer` : le dernier enregistré s'exécute en premier. Les nettoyages
s'exécutent **après** le corps du test, alors que la fonction utilitaire est terminée depuis
longtemps — impossible avec un `defer`, qui se serait exécuté à la sortie de `helper`. Les
`t.Log` des nettoyages apparaissent avec `-v` (ou en cas d'échec).

## Exercice intermédiaire — Tester `wordstat`

Le refactoring attendu, s'il n'était pas déjà fait :

```go
func main() {
	os.Exit(run(os.Args[1:], os.Stdin, os.Stdout, os.Stderr))
}

func run(args []string, stdin io.Reader, stdout, stderr io.Writer) int {
	fs := flag.NewFlagSet("wordstat", flag.ContinueOnError)
	fs.SetOutput(stderr)
	top := fs.Int("top", 10, "nombre de mots affichés")
	…
}
```

Chaque dépendance au monde extérieur (`os.Args`, `os.Stdin`, `os.Stdout`, `os.Stderr`,
`os.Exit`, `flag.CommandLine`) devient un **paramètre**. Le test de bout en bout :

```go
func TestRunStdin(t *testing.T) {
	var out, errOut bytes.Buffer
	code := run(nil, strings.NewReader("b a b"), &out, &errOut)
	if code != 0 {
		t.Fatalf("code = %d, attendu 0 ; stderr = %q", code, errOut.String())
	}
	if !strings.Contains(out.String(), "1. b") {
		t.Errorf("sortie = %q, attendu « b » en tête", out.String())
	}
}

func TestRunMissingFile(t *testing.T) {
	var out, errOut bytes.Buffer
	good := filepath.Join(t.TempDir(), "ok.txt")
	os.WriteFile(good, []byte("mot"), 0o644)
	code := run([]string{"absent.txt", good}, nil, &out, &errOut)
	if code != 1 {
		t.Errorf("code = %d, attendu 1 (S4)", code)
	}
	if !strings.Contains(errOut.String(), "absent.txt") {
		t.Errorf("stderr = %q, attendu une mention du fichier absent (S3)", errOut.String())
	}
	if !strings.Contains(out.String(), "mot") {
		t.Errorf("le second fichier n'a pas été traité (S3) : %q", out.String())
	}
}
```

`flag.ContinueOnError` est essentiel : avec la valeur par défaut `ExitOnError`, une option
invalide appellerait `os.Exit` **au milieu du test** et tuerait tout le processus de test.

## Défi — Le test qui ment

1. Trois implémentations fausses qui passent :
   ```go
   func ParsePort(s string) (int, error) { return 0, nil }        // ne fait rien
   func ParsePort(s string) (int, error) { return strconv.Atoi(s) } // accepte -1, 0, 99999
   func ParsePort(s string) (int, error) { return 80, nil }       // constante
   ```
2. Le test doit vérifier **la valeur** **et** les **refus** :
   ```go
   valid := map[string]int{"1": 1, "80": 80, "65535": 65535}
   for s, want := range valid {
   	got, err := ParsePort(s)
   	if err != nil || got != want {
   		t.Errorf("ParsePort(%q) = %d, %v ; attendu %d, nil", s, got, err, want)
   	}
   }
   for _, s := range []string{"", "0", "-1", "65536", "80a", " 80", "1e3"} {
   	if _, err := ParsePort(s); err == nil {
   		t.Errorf("ParsePort(%q) : erreur attendue", s)
   	}
   }
   ```
3. Manquaient : la vérification de la **valeur** (le `_ = p` est le cœur du mensonge), et tous
   les cas **invalides** : vide, zéro, négatif, juste au-dessus de la borne, non numérique,
   espaces, notation scientifique.
4. **Un test ne ment pas s'il échoue pour toute implémentation incorrecte qu'on peut
   raisonnablement écrire** — en pratique : il vérifie les résultats (pas seulement l'absence
   d'erreur), et il contient au moins autant de cas qui doivent échouer que de cas qui doivent
   réussir.

## Quiz

1. Le fichier se termine par `_test.go` ; la fonction s'appelle `TestXxx(t *testing.T)`.
2. Parce qu'un test est du code ordinaire ; un `if` avec un message écrit à la main est plus
   lisible et plus informatif qu'un DSL.
3. Obtenu, puis attendu : `F(x) = obtenu, attendu voulu`.
4. `Error` signale et continue, `Fatal` signale et arrête le test. `Fatal` quand la suite n'a plus
   de sens, `Error` sinon.
5. Elle exclut la fonction des positions rapportées : l'échec pointe la ligne de l'appelant.
6. `t.Cleanup` s'exécute à la fin du **test**, même enregistré depuis une fonction utilitaire ;
   `defer` à la fin de la **fonction** qui le contient.
7. `foo` : même paquet, accès à l'interne ; `foo_test` : paquet externe, API publique seulement.
8. Le résultat mémorisé d'une exécution précédente est réaffiché sans exécution ; `-count=1`.
9. `go test -run 'TestNom/sous_test'`.
10. À exécuter les tests dans un ordre aléatoire, pour révéler ceux qui dépendent d'un état
    laissé par un autre.
