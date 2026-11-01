# Leçon 7 — Fichiers et systèmes de fichiers

> Lire un fichier tient en une ligne. L'**écrire de façon sûre** — sans laisser un fichier à
> moitié écrit si le programme s'arrête, sans perdre l'erreur d'un disque plein, sans laisser
> un utilisateur lire `../../etc/passwd` — demande de connaître une dizaine de règles. C'est
> l'objet de cette leçon, qui s'appuie entièrement sur `io.Reader`/`io.Writer` (leçon 5).

## Objectifs

1. Ouvrir, lire, écrire et fermer des fichiers, en traitant **toutes** les erreurs.
2. Écrire de façon **atomique** (fichier temporaire + renommage).
3. Manipuler les chemins avec `path/filepath`, et se protéger de la traversée de répertoire avec `os.Root`.
4. Parcourir une arborescence, et abstraire le système de fichiers avec `io/fs`.
5. Embarquer des fichiers dans le binaire avec `//go:embed`.

## Explication

### Les fonctions d'un coup

```go
data, err := os.ReadFile("config.json")             // tout le fichier en mémoire
err = os.WriteFile("out.txt", data, 0o644)          // crée ou TRONQUE, puis écrit
entries, err := os.ReadDir(".")                     // triées par nom
```

Pratiques pour de petits fichiers dont la taille est maîtrisée. Pour un fichier de taille
inconnue, on ouvre et on traite en flux (leçon 5).

`0o644` est la **permission** Unix en octal (préfixe `0o`, Go 1.13) : lecture-écriture pour le
propriétaire, lecture pour les autres. `0o600` pour un fichier qui contient un secret.

### Ouvrir

| Fonction | Usage |
|---|---|
| `os.Open(name)` | lecture seule |
| `os.Create(name)` | écriture, **crée ou tronque** (perm. 0o666 avant umask) |
| `os.OpenFile(name, flags, perm)` | tout le reste : `os.O_APPEND`, `os.O_CREATE`, `os.O_EXCL`… |

```go
f, err := os.OpenFile("app.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0o644)
```

`os.O_EXCL` avec `O_CREATE` échoue si le fichier existe : la seule façon **sûre** de créer un
fichier « seulement s'il n'existe pas » (tester d'abord avec `os.Stat` puis créer laisse une
fenêtre où un autre processus peut le créer entre les deux).

### Fermer : lecture et écriture ne se traitent pas pareil

En **lecture**, l'erreur de `Close` n'apporte rien : `defer f.Close()` suffit.

En **écriture**, c'est différent : le système peut différer l'écriture réelle, et un disque plein
ou une erreur réseau (NFS) peut n'apparaître **qu'au `Close`**. Ignorer cette erreur, c'est
annoncer « sauvegardé » pour un fichier tronqué. Le patron :

```go
func save(name string, data []byte) (err error) {
	f, err := os.Create(name)
	if err != nil {
		return err
	}
	defer func() {
		if cerr := f.Close(); cerr != nil && err == nil {
			err = cerr   // résultat nommé : niveau 2, leçon 7
		}
	}()
	_, err = f.Write(data)
	return err
}
```

### Écriture atomique

`os.WriteFile` **tronque** le fichier puis écrit. Si le programme s'arrête au milieu (panne,
`kill`, disque plein), le fichier reste **à moitié écrit** — et l'ancienne version est perdue.
Pour un fichier de configuration ou une base de tâches, c'est inacceptable.

La solution est universelle : écrire dans un fichier **temporaire du même répertoire**, puis le
**renommer** sur la destination.

```
1. os.CreateTemp(dir, ".tasks-*.tmp")
2. écrire, puis f.Sync() (forcer l'écriture physique), puis f.Close()
3. os.Rename(tmp, "tasks.json")
```

Sur un système POSIX, `rename` dans un même système de fichiers est **atomique** : un lecteur
voit soit l'ancien fichier entier, soit le nouveau entier, jamais un mélange. « Même répertoire »
est essentiel : un `rename` entre deux systèmes de fichiers (`/tmp` est souvent en mémoire)
échoue ou n'est plus atomique. En cas d'erreur, supprimer le temporaire.

### Erreurs de fichiers

Les erreurs de `os` sont des `*fs.PathError` (opération, chemin, erreur sous-jacente) qui
enveloppent des sentinelles. On teste **avec `errors.Is`** (niveau 2, leçon 6) :

```go
_, err := os.Open("nope.txt")
errors.Is(err, fs.ErrNotExist)     // true
errors.Is(err, fs.ErrPermission)
errors.Is(err, fs.ErrExist)        // avec O_EXCL
```

`os.IsNotExist(err)` existe encore mais ne voit pas à travers un `%w` : préférer `errors.Is`.

### Chemins : `path/filepath`

| Fonction | Exemple | Résultat |
|---|---|---|
| `filepath.Join` | `Join("a", "../b", "/c")` | `b/c` — nettoie et utilise le bon séparateur |
| `filepath.Ext` | `Ext("x.tar.gz")` | `.gz` |
| `filepath.Base`, `Dir` | `Base("/a/b/")` | `b` |
| `filepath.Clean` | `Clean("a//b/./../c")` | `a/c` |
| `filepath.Abs`, `Rel` | | absolu, relatif |
| `filepath.IsLocal` (1.20) | `IsLocal("../x")` | `false` |

**Jamais** de concaténation avec `"/"` : le séparateur est `\` sous Windows. Le paquet `path`
(sans `file`) est pour les chemins **d'URL**, toujours en `/`.

### La traversée de répertoire et `os.Root`

Un serveur reçoit un nom de fichier d'un utilisateur et fait `os.Open(filepath.Join(baseDir,
name))`. Avec `name = "../../etc/passwd"`, `Join` nettoie le chemin… **et sort** de `baseDir`.
C'est la faille de *path traversal*, l'une des plus courantes du web.

`filepath.IsLocal` détecte les `..` et les chemins absolus, mais pas les **liens symboliques** :
un lien `uploads/evil → /etc/passwd` passe le test.

Go 1.24 a ajouté **`os.Root`**, qui confine **toutes** les opérations à un répertoire, liens
symboliques compris, au niveau du système :

```go
root, err := os.OpenRoot("uploads")
defer root.Close()

f, err := root.Open("../main.go")    // openat ../main.go: path escapes from parent
f, err := root.Open("evil")          // le lien vers /etc/passwd : refusé aussi
data, err := root.ReadFile("a.txt")  // ✓
```

`os.Root` offre `Open`, `Create`, `OpenFile`, `ReadFile`, `WriteFile`, `Mkdir`, `MkdirAll`,
`Remove`, `Stat`… Dès qu'un nom de fichier vient de l'extérieur : **`os.Root`**.

### Parcourir : `filepath.WalkDir`

```go
err := filepath.WalkDir("src", func(path string, d fs.DirEntry, err error) error {
	if err != nil {
		return err                  // répertoire illisible : on arrête (ou on continue : à choisir)
	}
	if d.IsDir() && d.Name() == ".git" {
		return filepath.SkipDir     // ne pas descendre
	}
	if !d.IsDir() && filepath.Ext(path) == ".go" {
		fmt.Println(path)
	}
	return nil
})
```

L'ordre est **lexical** (déterministe). Préférer `WalkDir` à l'ancien `Walk`, qui faisait un
`Stat` par fichier.

### `io/fs` : abstraire le système de fichiers

`fs.FS` est une interface d'**une** méthode, `Open(name string) (fs.File, error)` — dans
l'esprit d'`io.Reader`. Elle est satisfaite par :

- `os.DirFS("dir")` — un répertoire réel ;
- `embed.FS` — des fichiers embarqués dans le binaire ;
- `fstest.MapFS` — un système de fichiers **en mémoire**, pour les tests ;
- `zip.Reader` — le contenu d'une archive.

Les fonctions `fs.ReadFile`, `fs.WalkDir`, `fs.Glob`, `fs.Sub` marchent sur tous. Une fonction
qui accepte un `fs.FS` plutôt qu'un chemin se teste sans toucher au disque. Les chemins de
`fs.FS` utilisent **toujours** `/` et ne commencent pas par `/`.

### `//go:embed`

```go
import "embed"

//go:embed templates/*.html static
var assets embed.FS

//go:embed VERSION
var version string
```

La directive, placée juste au-dessus d'une variable de paquet, demande au compilateur d'inclure
les fichiers dans le binaire. Un `string` ou un `[]byte` pour un fichier unique, un `embed.FS`
pour plusieurs. Le binaire reste **autonome** (niveau 1, leçon 1) : pas de fichiers à déployer à
côté. Les fichiers commençant par `.` ou `_` sont exclus des répertoires, sauf avec `all:`.

### Fichiers temporaires

`os.CreateTemp(dir, pattern)` et `os.MkdirTemp(dir, pattern)` créent un nom **unique** (le `*`
du motif est remplacé). `dir = ""` utilise le répertoire temporaire du système. Toujours
nettoyer : `defer os.Remove(f.Name())`, `defer os.RemoveAll(dir)`. Le niveau 5 offrira
`t.TempDir()`, nettoyé automatiquement.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"io/fs"
	"os"
	"path/filepath"
	"strings"
)

// WriteAtomic remplace name par data sans jamais laisser de fichier à moitié écrit.
func WriteAtomic(name string, data []byte, perm fs.FileMode) (err error) {
	tmp, err := os.CreateTemp(filepath.Dir(name), "."+filepath.Base(name)+"-*.tmp")
	if err != nil {
		return err
	}
	defer func() {
		if err != nil {
			os.Remove(tmp.Name()) // nettoyage best effort en cas d'échec
		}
	}()
	if _, err = tmp.Write(data); err != nil {
		tmp.Close()
		return err
	}
	if err = tmp.Sync(); err != nil {
		tmp.Close()
		return err
	}
	if err = tmp.Close(); err != nil {
		return err
	}
	if err = os.Chmod(tmp.Name(), perm); err != nil {
		return err
	}
	return os.Rename(tmp.Name(), name)
}

// serveFile lit un fichier demandé par un utilisateur, confiné à root.
func serveFile(root *os.Root, name string) (string, error) {
	b, err := root.ReadFile(name)
	if err != nil {
		return "", fmt.Errorf("fichier %q : %w", name, err)
	}
	return strings.TrimSpace(string(b)), nil
}

func main() {
	dir, err := os.MkdirTemp("", "lecon07-*")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer os.RemoveAll(dir)

	public := filepath.Join(dir, "public")
	os.MkdirAll(filepath.Join(public, "docs"), 0o755)
	os.WriteFile(filepath.Join(dir, "secret.txt"), []byte("mot de passe"), 0o600)
	os.Symlink(filepath.Join(dir, "secret.txt"), filepath.Join(public, "raccourci"))

	cfg := filepath.Join(public, "docs", "config.json")
	for i := 1; i <= 2; i++ {
		if err := WriteAtomic(cfg, fmt.Appendf(nil, `{"version": %d}`, i), 0o644); err != nil {
			fmt.Println(err)
			return
		}
	}

	root, err := os.OpenRoot(public)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer root.Close()

	for _, name := range []string{"docs/config.json", "../secret.txt", "raccourci", "absent.txt"} {
		content, err := serveFile(root, name)
		switch {
		case errors.Is(err, fs.ErrNotExist):
			fmt.Printf("%-18s → 404\n", name)
		case err != nil:
			fmt.Printf("%-18s → refusé : %v\n", name, errors.Unwrap(err))
		default:
			fmt.Printf("%-18s → %s\n", name, content)
		}
	}

	// Aucun fichier temporaire ne doit traîner.
	filepath.WalkDir(public, func(p string, d fs.DirEntry, err error) error {
		if err != nil {
			return err
		}
		rel, _ := filepath.Rel(dir, p)
		fmt.Println("  ", rel)
		return nil
	})
}
```

Sortie :

```
docs/config.json   → {"version": 2}
../secret.txt      → refusé : openat ../secret.txt: path escapes from parent
raccourci          → refusé : openat raccourci: path escapes from parent
absent.txt         → 404
   public
   public/docs
   public/docs/config.json
   public/raccourci
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `CreateTemp(filepath.Dir(name), …)` | Le temporaire est dans le **même répertoire** que la cible : le `Rename` reste atomique. |
| `defer` + résultat nommé `err` | Une seule logique de nettoyage pour tous les chemins d'échec. |
| `Sync` puis `Close`, erreurs vérifiées | Sans `Sync`, une panne de courant juste après le `Rename` peut laisser un fichier vide ; l'erreur de `Close` peut révéler un disque plein. |
| `os.Chmod` | `CreateTemp` crée en `0o600` ; on applique la permission voulue avant de publier le fichier. |
| `fmt.Appendf` | Go 1.19 : formate directement dans un `[]byte`. |
| `root.ReadFile` | Le nom vient de l'utilisateur : **tout** passe par la racine confinée. |
| `raccourci` refusé | Le lien symbolique pointe hors de `public` : `IsLocal` l'aurait laissé passer, `os.Root` non. |
| `errors.Is(err, fs.ErrNotExist)` | À travers notre `%w` : `os.IsNotExist` aurait échoué. |
| Le parcours final | Aucun `.config.json-*.tmp` : les deux écritures atomiques ont bien renommé leur temporaire. |

## Erreurs fréquentes

1. **Ignorer l'erreur de `Close`** après une écriture.
2. **`os.WriteFile` sur un fichier important** : tronqué si le programme s'arrête.
3. **Temporaire dans `/tmp` puis `Rename`** vers un autre système de fichiers.
4. **`filepath.Join(base, userInput)`** sans `os.Root` : traversée de répertoire.
5. **Concaténer des chemins avec `"/"`**.
6. **`os.Stat` puis `os.Create`** pour « créer si absent » : condition de course ; `O_EXCL`.
7. **`os.IsNotExist` sur une erreur enveloppée.**
8. **`os.ReadFile` sur un fichier de taille non bornée.**
9. **Oublier de nettoyer** les fichiers et répertoires temporaires.
10. **Permissions 0o644 sur un fichier de secrets.**

## Bonnes pratiques Go

- Fonctions qui acceptent `io.Reader`/`io.Writer` ou `fs.FS` plutôt que des chemins.
- Écriture atomique pour tout fichier dont la perte serait grave.
- `os.Root` dès qu'un nom de fichier vient de l'extérieur.
- `errors.Is(err, fs.ErrNotExist)` pour tester les erreurs.
- `filepath` pour les chemins du système, `path` pour les URL.
- `//go:embed` pour livrer gabarits, fichiers statiques et migrations dans le binaire.

## Ce que je dois retenir

- `defer f.Close()` en lecture ; erreur de `Close` **vérifiée** en écriture.
- Écriture atomique : temporaire dans le **même répertoire**, `Sync`, `Close`, `Rename`.
- `fs.ErrNotExist`, `fs.ErrExist`, `fs.ErrPermission` avec `errors.Is`.
- `os.Root` (1.24) confine les accès, liens symboliques compris.
- `filepath.WalkDir`, `fs.FS`, `os.DirFS`, `fstest.MapFS`, `embed.FS`.
- `//go:embed` garde le binaire autonome.

---

➡️ [Exercices](exercices.md)
