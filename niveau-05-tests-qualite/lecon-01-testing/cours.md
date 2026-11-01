# Leçon 1 — Le paquet `testing`

> Go a un outil de test **intégré** : pas de framework à choisir, pas de bibliothèque
> d'assertions, pas de fichier de configuration. Un fichier `_test.go`, une fonction
> `TestXxx(t *testing.T)`, et `go test`. Cette sobriété est voulue — et elle surprend ceux qui
> viennent d'écosystèmes où le test est une affaire de framework. Cette leçon pose les bases
> que tout le niveau utilise.

## Objectifs

1. Écrire un test, le lancer, lire sa sortie.
2. Choisir entre `t.Error` et `t.Fatal`, et écrire un bon message d'échec.
3. Utiliser `t.Run`, `t.Helper`, `t.Cleanup`, `t.TempDir`, `t.Setenv`, `t.Context`.
4. Connaître les options de `go test` utilisées au quotidien.

## Explication

### Les conventions

- Les tests vivent dans des fichiers **`xxx_test.go`**, à côté du code testé. Ils ne sont pas
  compilés dans le binaire final.
- Une fonction de test s'appelle **`TestXxx`** (majuscule après `Test`) et prend un unique
  paramètre `t *testing.T`.
- `go test` compile le paquet **avec** ses fichiers de test et exécute toutes les `TestXxx`.

```go
// slug.go
package slug
…

// slug_test.go
package slug

import "testing"

func TestSlugLowercases(t *testing.T) {
	got := Slug("Bonjour")
	want := "bonjour"
	if got != want {
		t.Errorf("Slug(%q) = %q, attendu %q", "Bonjour", got, want)
	}
}
```

Il n'y a pas de fonction `assertEqual` : on écrit un `if`, et on dit ce qui ne va pas. C'est
un choix assumé de l'équipe Go : le test est du **code ordinaire**, qu'on lit sans connaître un
DSL, et le message d'échec est écrit par quelqu'un qui sait ce qui compte.

### Le message d'échec

Un bon message permet de comprendre l'échec **sans ouvrir le fichier de test**. La convention de
la bibliothèque standard :

```
Fonction(entrées) = obtenu, attendu valeur
```

```go
t.Errorf("Slug(%q) = %q, attendu %q", in, got, want)
// slug_test.go:12: Slug("Bonjour") = "Bonjour", attendu "bonjour"
```

Toujours **obtenu avant attendu**, toujours l'entrée, `%q` pour les chaînes (on voit les espaces
et les caractères invisibles — l'erreur récurrente « je n'ai pas vu l'espace final »).

### `Error` ou `Fatal`

| Méthode | Effet |
|---|---|
| `t.Error`, `t.Errorf` | signale l'échec et **continue** le test |
| `t.Fatal`, `t.Fatalf` | signale l'échec et **arrête** ce test immédiatement |
| `t.Log`, `t.Logf` | message affiché seulement en cas d'échec ou avec `-v` |
| `t.Skip`, `t.Skipf` | ignore le test (avec une raison) |

Règle : `Fatal` quand la suite du test **n'a plus de sens** (le fichier n'a pas pu être ouvert,
la valeur est nil et on va la déréférencer) ; `Error` dans tous les autres cas, pour voir
**tous** les problèmes en une seule exécution.

```go
cfg, err := Load(path)
if err != nil {
	t.Fatalf("Load(%q) : %v", path, err)   // inutile de vérifier les champs de cfg
}
if cfg.Port != 8080 {
	t.Errorf("Port = %d, attendu 8080", cfg.Port)
}
if cfg.Debug {
	t.Error("Debug = true, attendu false")  // on veut aussi voir cette erreur-là
}
```

`Fatal` arrête le **test courant** (via `runtime.Goexit`), pas les autres. Et il ne doit être
appelé que depuis la goroutine du test (niveau 6).

### Les sous-tests : `t.Run`

```go
func TestSlug(t *testing.T) {
	t.Run("minuscules", func(t *testing.T) { … })
	t.Run("accents", func(t *testing.T) { … })
}
```

Chaque sous-test a un nom (`TestSlug/accents`), réussit ou échoue indépendamment, et peut être
lancé seul : `go test -run 'TestSlug/accents'`. Un `t.Fatal` dans un sous-test n'arrête que
lui. C'est la base du patron de la leçon 2.

### `t.Helper`

Une fonction utilitaire qui appelle `t.Errorf` ferait pointer l'échec sur **sa** ligne, pas sur
celle du test qui l'appelle. `t.Helper()` corrige cela :

```go
func assertSlug(t *testing.T, in, want string) {
	t.Helper()                     // l'échec sera rapporté à la ligne de l'APPELANT
	if got := Slug(in); got != want {
		t.Errorf("Slug(%q) = %q, attendu %q", in, got, want)
	}
}
```

### Ressources : `t.TempDir`, `t.Cleanup`, `t.Setenv`, `t.Context`

```go
dir := t.TempDir()                       // répertoire unique, SUPPRIMÉ à la fin du test
t.Setenv("TODO_FILE", filepath.Join(dir, "t.json"))   // restauré à la fin du test
t.Cleanup(func() { srv.Close() })        // exécuté à la fin, comme un defer, en ordre LIFO
ctx := t.Context()                       // Go 1.24 : annulé juste avant les Cleanup (niveau 6)
```

`t.Cleanup` diffère d'un `defer` sur un point : il peut être enregistré **dans une fonction
utilitaire** et s'exécutera à la fin du **test**, pas à la fin de la fonction utilitaire. Une
fonction `newTestServer(t)` peut ainsi se charger de son propre nettoyage.

### Tests de paquet et tests externes

Le fichier de test peut déclarer :

- `package slug` — même paquet : accès aux identifiants **non exportés** (tests « boîte blanche ») ;
- `package slug_test` — paquet **externe** (la seule exception à la règle « un répertoire = un
  paquet », vue au niveau 1) : on teste l'API **publique**, comme un utilisateur.

Préférer `package xxx_test` pour l'API publique : si un test externe est difficile à écrire,
l'API est difficile à utiliser. Les deux peuvent coexister dans le même répertoire.

### `go test` au quotidien

| Commande | Effet |
|---|---|
| `go test ./...` | tous les paquets du module |
| `go test -v` | affiche chaque test et les `t.Log` |
| `go test -run 'Slug/accents'` | filtre par expression régulière (sur `/`-séparés) |
| `go test -count=1` | désactive le **cache** des résultats |
| `go test -race` | active le détecteur de *data races* (niveau 6) |
| `go test -short` | `testing.Short()` rend `true` : sauter les tests lents |
| `go test -failfast` | s'arrête au premier test en échec |
| `go test -shuffle=on` | ordre aléatoire : révèle les tests qui dépendent les uns des autres |

**Le cache** : `go test ./...` ne relance pas un paquet dont ni le code ni les tests n'ont
changé, et affiche `(cached)`. Si un test lit un fichier ou une variable d'environnement non
déclarés, le cache peut masquer un changement : `-count=1` force l'exécution.

Depuis Go 1.27, `go test` lance aussi l'analyseur `stdversion` (en plus d'un sous-ensemble de
`go vet`) : il signale l'usage d'une API plus récente que la version déclarée dans `go.mod`.

### Ce qu'on teste

- Le **comportement observable** (entrées → sorties, effets), pas l'implémentation : un test qui
  casse à chaque refactorisation sans changement de comportement est un fardeau.
- Les **cas limites** : vide, un seul élément, valeurs extrêmes, Unicode, erreurs.
- Chaque **bug corrigé** : d'abord un test qui le reproduit (et échoue), puis la correction.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : wordcount/wordcount.go
// Package wordcount compte les mots d'un texte.
package wordcount

import (
	"bufio"
	"io"
	"os"
	"strings"
	"unicode"
)

// Count rend le nombre d'occurrences de chaque mot, en minuscules,
// ponctuation retirée aux bords.
func Count(r io.Reader) (map[string]int, error) {
	counts := map[string]int{}
	sc := bufio.NewScanner(r)
	sc.Split(bufio.ScanWords)
	for sc.Scan() {
		w := strings.ToLower(strings.TrimFunc(sc.Text(), unicode.IsPunct))
		if w != "" {
			counts[w]++
		}
	}
	return counts, sc.Err()
}

// CountFile applique Count au contenu d'un fichier.
func CountFile(path string) (map[string]int, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer f.Close()
	return Count(f)
}
```

```go
// Fichier : wordcount/wordcount_test.go
package wordcount_test

import (
	"errors"
	"io/fs"
	"os"
	"path/filepath"
	"strings"
	"testing"

	"exemple/wordcount"
)

// assertCount vérifie un mot ; t.Helper fait pointer l'échec sur l'appelant.
func assertCount(t *testing.T, got map[string]int, word string, want int) {
	t.Helper()
	if got[word] != want {
		t.Errorf("count[%q] = %d, attendu %d", word, got[word], want)
	}
}

func TestCount(t *testing.T) {
	t.Run("casse et ponctuation", func(t *testing.T) {
		got, err := wordcount.Count(strings.NewReader("Le chat, le CHAT ; le chien."))
		if err != nil {
			t.Fatalf("Count : erreur inattendue %v", err)
		}
		assertCount(t, got, "le", 3)
		assertCount(t, got, "chat", 2)
		assertCount(t, got, "chien", 1)
	})

	t.Run("texte vide", func(t *testing.T) {
		got, err := wordcount.Count(strings.NewReader(""))
		if err != nil {
			t.Fatalf("Count : %v", err)
		}
		if len(got) != 0 {
			t.Errorf("Count(\"\") = %v, attendu une map vide", got)
		}
	})

	t.Run("ponctuation seule", func(t *testing.T) {
		got, _ := wordcount.Count(strings.NewReader("... ! ?"))
		if len(got) != 0 {
			t.Errorf("Count(%q) = %v, attendu une map vide", "... ! ?", got)
		}
	})
}

func TestCountFile(t *testing.T) {
	dir := t.TempDir() // supprimé automatiquement à la fin du test
	path := filepath.Join(dir, "texte.txt")
	if err := os.WriteFile(path, []byte("été été hiver"), 0o644); err != nil {
		t.Fatal(err)
	}

	got, err := wordcount.CountFile(path)
	if err != nil {
		t.Fatalf("CountFile(%q) : %v", path, err)
	}
	assertCount(t, got, "été", 2)

	_, err = wordcount.CountFile(filepath.Join(dir, "absent.txt"))
	if !errors.Is(err, fs.ErrNotExist) {
		t.Errorf("CountFile(absent) : erreur = %v, attendu fs.ErrNotExist", err)
	}
}
```

Sortie de `go test -v ./...` :

```
=== RUN   TestCount
=== RUN   TestCount/casse_et_ponctuation
=== RUN   TestCount/texte_vide
=== RUN   TestCount/ponctuation_seule
--- PASS: TestCount (0.00s)
    --- PASS: TestCount/casse_et_ponctuation (0.00s)
    --- PASS: TestCount/texte_vide (0.00s)
    --- PASS: TestCount/ponctuation_seule (0.00s)
=== RUN   TestCountFile
--- PASS: TestCountFile (0.00s)
PASS
ok  	exemple/wordcount	0.003s
```

Et si l'on **casse** le code (retirer le `strings.ToLower`) :

```
--- FAIL: TestCount (0.00s)
    --- FAIL: TestCount/casse_et_ponctuation (0.00s)
        wordcount_test.go:28: count["le"] = 2, attendu 3
        wordcount_test.go:29: count["chat"] = 1, attendu 2
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `package wordcount_test` | Test **externe** : seule l'API publique est utilisée, via l'import `exemple/wordcount`. |
| `Count(r io.Reader)` | La fonction testable reçoit un `Reader` : le test passe une chaîne, pas un fichier. `CountFile` n'est qu'une fine couche. |
| `t.Fatalf` sur l'erreur | Si `Count` échoue, vérifier les comptes n'a pas de sens. |
| `assertCount` + `t.Helper()` | En cas d'échec, la ligne rapportée (`:28`, `:29`) est celle du test, pas celle de l'utilitaire. |
| `t.Error` dans `assertCount` | Les deux échecs sont rapportés dans la même exécution. |
| Noms de sous-tests | Les espaces deviennent `_` : `TestCount/casse_et_ponctuation`. |
| `t.TempDir()` | Aucun nettoyage à écrire, aucun fichier qui traîne si le test échoue. |
| `errors.Is(err, fs.ErrNotExist)` | On teste la **nature** de l'erreur, jamais son texte. |

## Erreurs fréquentes

1. **Un test qui ne peut pas échouer** : jamais vérifié en cassant le code.
2. **Message sans l'entrée ni l'obtenu** : `t.Error("mauvais résultat")`.
3. **`t.Fatal` partout** : on ne voit qu'un problème à la fois.
4. **Comparer des messages d'erreur** (`err.Error() == "…"`) au lieu de `errors.Is`/`As`.
5. **Des tests qui dépendent les uns des autres** (ordre, état global, fichier partagé).
6. **Des fichiers temporaires créés à la main** et jamais supprimés.
7. **Oublier `t.Helper()`** : les échecs pointent tous la même ligne de l'utilitaire.
8. **Tester l'implémentation** (appels internes, champs privés) plutôt que le comportement.
9. **Croire qu'un test est passé** alors qu'il était `(cached)`.

## Bonnes pratiques Go

- Un fichier `xxx_test.go` par fichier `xxx.go`, quand c'est naturel.
- `package xxx_test` par défaut ; `package xxx` quand on doit atteindre l'interne.
- Messages `F(entrée) = obtenu, attendu voulu`, avec `%q` pour les chaînes.
- `Error` par défaut, `Fatal` quand la suite n'a plus de sens.
- `t.TempDir`, `t.Setenv`, `t.Cleanup` plutôt que des nettoyages manuels.
- Lancer `go test -race -shuffle=on ./...` avant chaque commit.
- **Casser le code pour voir le test échouer**, au moins une fois.

## Ce que je dois retenir

- `xxx_test.go`, `func TestXxx(t *testing.T)`, `go test ./...`.
- Pas d'assertions : un `if` et un message `obtenu, attendu`.
- `Error` continue, `Fatal` arrête le test courant.
- `t.Run` pour les sous-tests ; `-run 'Test/sous'` pour en lancer un.
- `t.Helper`, `t.TempDir`, `t.Cleanup`, `t.Setenv`, `t.Context`.
- `package xxx_test` teste l'API publique.
- `-count=1` contre le cache, `-race`, `-shuffle=on`.

---

➡️ [Exercices](exercices.md)
