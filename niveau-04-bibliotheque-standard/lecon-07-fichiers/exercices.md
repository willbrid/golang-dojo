# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-04/lecon-07/`. Tous les fichiers de test sont créés dans un
répertoire temporaire supprimé à la fin.

## Exercices faciles

**E1 — Les erreurs de fichiers.**
Provoquer et afficher (avec `%v` **et** `%T`) : un fichier inexistant, un fichier sans droit de
lecture (`chmod 000`), un `OpenFile` avec `O_CREATE|O_EXCL` sur un fichier existant, un
`ReadFile` sur un répertoire. Pour chacun, quelle sentinelle `fs.Err…` reconnaît-on avec
`errors.Is` ?

**E2 — `IsNotExist` contre `errors.Is`.**
Envelopper l'erreur d'un `os.Open` raté avec `fmt.Errorf("config : %w", err)`. Tester avec
`os.IsNotExist` puis avec `errors.Is(err, fs.ErrNotExist)`. Expliquer la différence avec la doc
(`go doc os.IsNotExist`).

**E3 — La troncature.**
Écrire un fichier de 1 Mo avec `os.WriteFile`, puis lancer un programme qui le réécrit avec
`os.WriteFile` et s'interrompt (`os.Exit(1)`) **au milieu** (écrire en deux `Write` via
`os.Create` pour simuler). Quelle taille a le fichier ? Recommencer avec `WriteAtomic` du cours.

**E4 — Traversée.**
Pour chacun de ces noms fournis par un « utilisateur », dire ce que donnent
`filepath.Join("public", name)`, `filepath.IsLocal(name)` et `root.Open(name)` :
`a.txt`, `../a.txt`, `docs/../../a.txt`, `/etc/passwd`, `docs/./a.txt`, et un lien symbolique
`lien → ../secret`. Présenter en tableau.

**E5 — `fs.FS` en mémoire.**
Écrire `CountGoLines(fsys fs.FS) (int, error)` qui compte les lignes de tous les `.go` d'une
arborescence, avec `fs.WalkDir`. L'appeler sur `os.DirFS(".")` **et** sur un `fstest.MapFS`
construit à la main. Quel avantage pour un test ?

---

## Exercice intermédiaire — `dedup`, trouver les fichiers en double

Écrire `dedup DIR` qui liste les groupes de fichiers au contenu **identique** dans une
arborescence.

**Contraintes :**
1. Ne lire le contenu que des fichiers de **même taille** (la taille vient de `d.Info()`) : la
   plupart des fichiers sont éliminés sans être ouverts.
2. Comparer par empreinte SHA-256 calculée **en flux** (`io.Copy` dans un `hash.Hash`).
3. Ignorer les liens symboliques et les répertoires `.git`.
4. Un fichier illisible est signalé sur `os.Stderr` et n'interrompt pas le parcours.
5. Sortie déterministe : groupes triés par taille décroissante, chemins triés dans chaque groupe.
6. Afficher l'espace récupérable total.
7. **Option `-delete`** : supprimer les doublons en gardant le premier chemin (ordre
   alphabétique) de chaque groupe. Avant d'écrire cette option, lister par écrit **trois**
   façons dont elle pourrait détruire des données à tort, et comment chacune est évitée.

---

## Défi — Une base de tâches qui survit à tout

Préparation directe du projet 2.

```go
type Store struct { /* chemin du fichier */ }
func Open(path string) (*Store, error)
func (s *Store) Load() ([]Task, error)
func (s *Store) Save(tasks []Task) error
```

1. `Save` est atomique. `Load` sur un fichier absent rend une liste vide, pas une erreur.
2. Deux processus lancés en même temps qui font `Load` → modification → `Save` : que se passe-t-il ?
   (On appelle ce problème la *mise à jour perdue*.) Le démontrer avec deux terminaux et un
   `time.Sleep` entre `Load` et `Save`.
3. Proposer une parade **sans** dépendance externe. *(Pistes : un fichier verrou créé avec
   `O_EXCL` ; un numéro de version vérifié au `Save`.)* Implémenter l'une des deux, et dire ce
   qui se passe si le processus meurt en tenant le verrou.
4. Garder les 3 dernières sauvegardes (`tasks.json.1`, `.2`, `.3`) par rotation. Dans quel ordre
   faire les renommages pour qu'une interruption à n'importe quel moment ne perde jamais la
   version courante ?

---

## Quiz

1. Pourquoi faut-il vérifier l'erreur de `Close` après une écriture, mais pas après une lecture ?
2. Quelles sont les étapes d'une écriture atomique ? Pourquoi le temporaire doit-il être dans le même répertoire ?
3. À quoi sert `f.Sync()` ?
4. Pourquoi `os.Stat` puis `os.Create` n'est-il pas sûr pour « créer seulement si absent » ?
5. Comment tester qu'une erreur signifie « fichier inexistant » ?
6. Que protège `os.Root` que `filepath.IsLocal` ne protège pas ?
7. Différence entre les paquets `path` et `path/filepath` ?
8. Qu'est-ce que `fs.FS`, et pourquoi accepter un `fs.FS` plutôt qu'un chemin ?
9. Que fait `//go:embed`, et quel type de variable accepte-t-il ?
10. Quelle permission donner à un fichier contenant un secret ?
