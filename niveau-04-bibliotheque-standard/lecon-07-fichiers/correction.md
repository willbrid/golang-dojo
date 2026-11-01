# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Les erreurs de fichiers

Toutes sont des `*fs.PathError` :

| Situation | Message | Sentinelle |
|---|---|---|
| fichier inexistant | `open …/nope: no such file or directory` | `fs.ErrNotExist` |
| `chmod 000` | `open …/f: permission denied` | `fs.ErrPermission` |
| `O_CREATE\|O_EXCL` sur un existant | `open …/f: file exists` | `fs.ErrExist` |
| `ReadFile` sur un répertoire | `read …: is a directory` | **aucune** des trois |

Le dernier cas est utile : toutes les erreurs n'ont pas de sentinelle. Pour lui, `errors.As`
vers `*fs.PathError` donne l'opération (`read`) et le chemin ; l'erreur sous-jacente est
`syscall.EISDIR`. Attention : si les tests tournent en `root` (conteneur), `chmod 000` ne bloque
rien — le cas « permission » ne se reproduit pas.

## E2 — `IsNotExist` contre `errors.Is`

Mesuré : `os.IsNotExist(w)` → `false`, `errors.Is(w, fs.ErrNotExist)` → `true`.

La documentation le dit : *« This function predates errors.Is. It only supports errors returned
by the os package. New code should use errors.Is(err, fs.ErrNotExist). »* `IsNotExist` regarde
l'erreur **elle-même** ; `errors.Is` remonte la chaîne des `%w` (N2 L6).

## E3 — La troncature

Après un `os.Create` suivi d'un seul des deux `Write` (1 Mo prévu, 512 Ko écrits) puis
`os.Exit(1)` : le fichier fait **512 Ko**, l'ancienne version (1 Mo) est **perdue**. `os.Create`
a tronqué à zéro dès l'ouverture.

Avec `WriteAtomic` interrompu au même endroit : la cible garde son contenu d'origine intact ; il
reste un `.nom-123456.tmp` orphelin dans le répertoire. C'est le compromis : un temporaire
oublié (qu'un nettoyage au démarrage peut supprimer) plutôt qu'une donnée corrompue.

## E4 — Traversée

| Nom | `filepath.Join("public", nom)` | `IsLocal` | `root.Open` |
|---|---|---|---|
| `a.txt` | `public/a.txt` | true | ✓ |
| `../a.txt` | `a.txt` — **hors de public** | false | `path escapes from parent` |
| `docs/../../a.txt` | `a.txt` — **hors de public** | false | `path escapes from parent` |
| `/etc/passwd` | `public/etc/passwd` | false | `path escapes from parent` |
| `docs/./a.txt` | `public/docs/a.txt` | true | ✓ |
| `lien` → `../secret` | `public/lien` | **true** | `path escapes from parent` |

Deux enseignements : `Join` ne « protège » pas — il **calcule** un chemin qui peut sortir de la
base ; et `IsLocal` est purement lexical — il ne voit pas où mène un lien symbolique. Seul
`os.Root`, qui résout chaque composant **côté système** (`openat`), refuse les six cas dangereux.

## E5 — `fs.FS` en mémoire

```go
func CountGoLines(fsys fs.FS) (int, error) {
	total := 0
	err := fs.WalkDir(fsys, ".", func(p string, d fs.DirEntry, err error) error {
		if err != nil || d.IsDir() || path.Ext(p) != ".go" {
			return err
		}
		f, err := fsys.Open(p)
		if err != nil {
			return err
		}
		defer f.Close()
		sc := bufio.NewScanner(f)
		for sc.Scan() {
			total++
		}
		return sc.Err()
	})
	return total, err
}

mem := fstest.MapFS{
	"main.go":       {Data: []byte("package main\n\nfunc main() {}\n")},
	"pkg/a.go":      {Data: []byte("package pkg\n")},
	"pkg/notes.txt": {Data: []byte("ignoré\n")},
}
n, err := CountGoLines(mem) // 4
```

Noter `path.Ext` et non `filepath.Ext` : dans un `fs.FS`, les chemins sont **toujours** en `/`.
L'avantage pour un test : aucun fichier à créer ni à nettoyer, des cas limites (fichier vide,
arborescence profonde) construits en une ligne, et des tests rapides et parallélisables.

## Exercice intermédiaire — `dedup`

```go
func hashFile(path string) ([sha256.Size]byte, error) {
	var sum [sha256.Size]byte
	f, err := os.Open(path)
	if err != nil {
		return sum, err
	}
	defer f.Close()
	h := sha256.New()
	if _, err := io.Copy(h, f); err != nil {
		return sum, err
	}
	copy(sum[:], h.Sum(nil))
	return sum, nil
}

func findDuplicates(dir string) ([][]string, int64, error) {
	bySize := map[int64][]string{}
	err := filepath.WalkDir(dir, func(p string, d fs.DirEntry, err error) error {
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			return nil // illisible : signalé, on continue
		}
		if d.IsDir() && d.Name() == ".git" {
			return filepath.SkipDir
		}
		if !d.Type().IsRegular() { // répertoires, liens symboliques, périphériques
			return nil
		}
		info, err := d.Info()
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			return nil
		}
		bySize[info.Size()] = append(bySize[info.Size()], p)
		return nil
	})
	if err != nil {
		return nil, 0, err
	}

	var groups [][]string
	var wasted int64
	for size, paths := range bySize {
		if len(paths) < 2 || size == 0 {
			continue
		}
		byHash := map[[sha256.Size]byte][]string{}
		for _, p := range paths {
			sum, err := hashFile(p)
			if err != nil {
				fmt.Fprintln(os.Stderr, err)
				continue
			}
			byHash[sum] = append(byHash[sum], p)
		}
		for _, g := range byHash {
			if len(g) > 1 {
				slices.Sort(g)
				groups = append(groups, g)
				wasted += size * int64(len(g)-1)
			}
		}
	}
	sizeOf := func(p string) int64 { fi, _ := os.Stat(p); return fi.Size() }
	slices.SortFunc(groups, func(a, b []string) int {
		return cmp.Or(cmp.Compare(sizeOf(b[0]), sizeOf(a[0])), cmp.Compare(a[0], b[0]))
	})
	return groups, wasted, nil
}

```

Sur une arborescence de test (avec un lien symbolique et un `.git`) :

```
[e x/f]
[a x/b]
17
```

Points de correction :

- **Deux passes** : par taille d'abord (gratuit, `d.Info()`), puis par empreinte seulement dans
  les groupes de même taille. Sur un vrai disque, l'immense majorité des fichiers n'est jamais
  ouverte.
- **`d.Type().IsRegular()`** écarte en une condition les répertoires, liens symboliques, tubes et
  périphériques. `WalkDir` ne suit pas les liens symboliques.
- **Clé de map `[32]byte`** : un **tableau** est comparable, il peut servir de clé ; un `[]byte`
  non (N1 L7). D'où le `copy(sum[:], h.Sum(nil))`.
- Fichiers vides exclus : ils sont tous « identiques » et ne libèrent rien.
- `fmt.Fprintln(os.Stderr, err)` puis `return nil` : l'erreur est **journalisée et absorbée**,
  conformément à la contrainte 4 — on ne la retourne pas en plus.
- Le tri appelle `os.Stat` dans le comparateur : correct mais coûteux ; garder la taille dans une
  struct `group{size int64; paths []string}` serait plus propre.

**Trois façons dont `-delete` peut détruire à tort :**

1. **Collision d'empreinte** : astronomiquement improbable avec SHA-256, mais un outil de
   suppression peut comparer **octet par octet** avant de supprimer — coût faible, certitude totale.
2. **Liens physiques** (*hard links*) : deux chemins, **un seul** fichier. « Supprimer le doublon »
   ne libère rien… et c'est sans danger ; mais l'inverse — un même fichier vu deux fois via deux
   points de montage — peut conduire à supprimer l'unique copie. Comparer l'identité système
   (`os.SameFile`) avant suppression.
3. **Le fichier change entre l'analyse et la suppression** : recalculer l'empreinte juste avant
   de supprimer. Et toujours proposer un mode `-n` (*dry run*) qui affiche sans agir.

## Défi — Une base de tâches qui survit à tout

1. `Save` reprend `WriteAtomic` ; `Load` teste `errors.Is(err, fs.ErrNotExist)` et rend
   `nil, nil`.
2. **Mise à jour perdue** : A lit `[t1]`, B lit `[t1]`, A sauve `[t1, t2]`, B sauve `[t1, t3]`.
   Résultat : `[t1, t3]` — `t2` a disparu, sans aucune erreur. L'atomicité du fichier ne protège
   pas l'atomicité de la **séquence** lecture-modification-écriture.
3. Deux parades :
   - **Verrou** : `os.OpenFile("tasks.lock", O_CREATE|O_EXCL|O_WRONLY, 0o600)` ; `O_EXCL` garantit
     qu'un seul processus le crée. On écrit son PID dedans, on le supprime en fin. Si le processus
     meurt en le tenant, le verrou reste : il faut une règle de récupération (verrou plus vieux
     que N secondes, PID qui n'existe plus) — et cette règle est elle-même délicate.
   - **Version optimiste** : le fichier contient un champ `version`. `Save` relit la version sur
     disque juste avant de renommer ; si elle a changé depuis le `Load`, erreur « modifié par un
     autre processus, recharger ». Pas de verrou orphelin possible, mais une petite fenêtre reste
     entre la vérification et le renommage. C'est le principe du verrouillage optimiste des bases
     de données (niveau 8, leçon 4), qui y est fait correctement.
4. **Rotation sûre** : renommer du plus ancien au plus récent — `.2 → .3`, `.1 → .2`, courant
   → `.1` **par copie** (pas par renommage), puis écrire le nouveau courant de façon atomique.
   Ainsi, à tout instant, le fichier courant existe. Renommer le courant en `.1` laisserait une
   fenêtre sans fichier courant.

## Quiz

1. En écriture, le système peut différer l'écriture réelle : une erreur (disque plein) peut
   n'apparaître qu'au `Close`. En lecture, il n'y a rien à perdre.
2. Créer un temporaire, écrire, `Sync`, `Close`, `Rename`. Même répertoire = même système de
   fichiers, condition de l'atomicité du `rename`.
3. Forcer l'écriture des données sur le support physique (sinon une panne peut les perdre).
4. Un autre processus peut créer le fichier entre les deux appels ; `O_EXCL` fait les deux en
   une opération atomique.
5. `errors.Is(err, fs.ErrNotExist)`.
6. Les liens symboliques qui pointent hors de la racine (et les courses sur les chemins).
7. `path` : chemins à `/` (URL, `fs.FS`) ; `filepath` : chemins du système d'exploitation.
8. Une interface d'une méthode (`Open`) qui abstrait un système de fichiers ; elle permet de
   passer un répertoire réel, des fichiers embarqués ou un système en mémoire pour les tests.
9. Il inclut des fichiers dans le binaire à la compilation ; `string`, `[]byte` ou `embed.FS`.
10. `0o600`.
