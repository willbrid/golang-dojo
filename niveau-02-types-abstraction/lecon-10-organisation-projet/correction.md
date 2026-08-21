# Leçon 10 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — `main` + `run`

```go
func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, "erreur :", err)
		os.Exit(1)
	}
}

func run() error {
	defer fmt.Println("nettoyage")   // S'EXÉCUTE, même en cas d'erreur
	// …
	return errors.New("échec simulé")
}
```

```
nettoyage
erreur : échec simulé
```

Le `defer` s'exécute parce que `run` **retourne** normalement — l'erreur est une valeur, pas un
saut. Si `os.Exit(1)` avait été appelé à l'intérieur de `run`, le `defer` aurait été
**purement et simplement ignoré** : `os.Exit` termine le processus sans dérouler la pile.

`grep -rn "os.Exit"` doit retourner **une seule ligne**. C'est un audit qui prend deux secondes
et détecte immédiatement une fuite de ressources potentielle.

## E2 — Entrées/sorties en paramètres

```go
func run(args []string, stdout, stderr io.Writer) error {
	fmt.Fprintln(stdout, "résultat")
	return nil
}

func main() {
	if err := run(os.Args[1:], os.Stdout, os.Stderr); err != nil { … }

	// Le même programme, capturé
	var buf bytes.Buffer
	_ = run([]string{"--json"}, &buf, io.Discard)
	fmt.Println(strings.ToUpper(buf.String()))
}
```

**Ce qu'on vient de rendre possible : le test.**

```go
func TestRun(t *testing.T) {
	var out bytes.Buffer
	if err := run([]string{"42"}, &out, io.Discard); err != nil {
		t.Fatal(err)
	}
	if got := out.String(); got != "résultat\n" {
		t.Errorf("sortie = %q", got)
	}
}
```

Aucun processus lancé, aucun fichier temporaire, aucune capture de descripteur — juste un appel
de fonction. C'est **la** raison de ce patron, et elle vaut bien les trois paramètres
supplémentaires. Le niveau 5 l'exploitera systématiquement.

`io.Writer` plutôt que `*os.File` : le paramètre accepte alors un buffer, un fichier, une
connexion réseau ou `io.Discard`. C'est « accept interfaces » appliqué à `main`.

## E3 — Les quatre stades

| Stade | Gagné | Perdu |
|---|---|---|
| 1 fichier | navigation instantanée, aucune cérémonie | difficile à lire au-delà de ~300 lignes |
| N fichiers, 1 package | lisibilité par thème, aucun export forcé | rien |
| N packages | frontières réelles, testabilité isolée | exports obligatoires, risque de cycles, friction |
| N binaires | plusieurs points d'entrée partageant le cœur | un niveau de chemin de plus |

**Pour un programme de 300 lignes, le stade 2 est le bon.** Passer au stade 3 obligerait à
exporter des identifiants pour rien : chaque frontière de package transforme un détail
d'implémentation en API. Le stade 3 se justifie quand une partie du code mérite un
**vocabulaire propre** ou doit être testée indépendamment.

## E4 — `internal/` en pratique

```
monprojet/internal/core   → depuis un autre module : use of internal package not allowed
monprojet/pkg/api         → depuis un autre module : fonctionne parfaitement
```

Les deux fonctionnent depuis `monprojet` lui-même.

**Ce que `pkg/` a apporté de vérifiable par le compilateur : rien.** C'est un répertoire au nom
conventionnel, strictement équivalent à `api/` ou `lib/` du point de vue de l'outillage. Il
allonge simplement tous les chemins d'import d'un segment.

`internal/`, lui, est un **mécanisme du langage**, appliqué par le compilateur. C'est la seule
frontière de visibilité au-delà du package.

## E5 — Couche contre domaine

Changement demandé : « une commande doit vérifier le crédit de l'utilisateur ».

**Découpage par couche** — `services/order.go` doit appeler `services/user.go`, qui manipule
`models/user.go`, lequel a besoin de connaître `models/order.go` pour calculer l'encours.
**Trois packages touchés**, et un **cycle apparaît** entre `models/user` et `models/order` dès
que la relation devient bidirectionnelle. On le casse en ajoutant des identifiants au lieu des
types, ce qui appauvrit le modèle.

**Découpage par domaine** — `order/service.go` définit chez lui l'interface dont il a besoin :
```go
type creditChecker interface {
	AvailableCredit(userID string) (int64, error)
}
```
`user` l'implémente sans le savoir. **Un package touché**, plus une interface de trois lignes.
**Aucun cycle possible** : la dépendance va de `order` vers `user`, jamais l'inverse.

C'est exactement l'expérience qui convainc : sur le papier les deux découpages se valent, à la
première évolution transverse ils divergent radicalement.

## Exercice intermédiaire — restructurer `taskman`

```
taskman/
├── go.mod
├── README.md                  graphe de dépendances + justification
├── cmd/
│   ├── taskman/main.go        ~25 lignes
│   └── taskman-export/main.go ~25 lignes
└── internal/
    ├── task/                  LE DOMAINE — stdlib uniquement
    │   ├── task.go
    │   ├── list.go
    │   └── repository.go      l'INTERFACE dont le domaine a besoin
    ├── storage/
    │   ├── json.go            implémente task.Repository
    │   └── dryrun.go          implémente task.Repository, n'écrit rien
    ├── config/
    │   └── config.go          lecture de l'environnement
    └── render/
        ├── table.go
        └── csv.go
```

**Graphe (acyclique) :**
```
cmd/taskman ──┬──→ config
              ├──→ storage ──→ task
              ├──→ render
              └──→ task
```

**Contrainte 2 — pourquoi `cmd/` devient justifié :** il y a maintenant **deux binaires**. Avec
un seul `main.go` à la racine, le second n'aurait nulle part où aller sans créer une asymétrie.

**Contrainte 3 — la vérification :**
```bash
$ go list -deps ./internal/task | grep -v '^internal/\|^\w*$' | grep '\.'
# aucune sortie : que de la bibliothèque standard
```
C'est la **définition opérationnelle** d'un domaine pur : il ne connaît ni JSON, ni
l'environnement, ni la ligne de commande. On peut le réutiliser derrière une API HTTP sans
toucher une ligne.

**Contrainte 6 — `--dry-run` sans `if` disséminés.** Le mot attendu est **interface**.

```go
// package task — l'interface est définie CÔTÉ CONSOMMATEUR
type Repository interface {
	Load() ([]Task, error)
	Save([]Task) error
}
```

```go
// package storage
type JSONStore struct{ path string }
func (s *JSONStore) Save(ts []task.Task) error { /* écrit réellement */ }

// dryRunStore décore : il lit vraiment, mais n'écrit jamais.
type dryRunStore struct {
	task.Repository                  // embarquée : Load est PROMUE
	out io.Writer
}
func (d dryRunStore) Save(ts []task.Task) error {
	fmt.Fprintf(d.out, "[dry-run] %d tâches auraient été écrites\n", len(ts))
	return nil
}
```

```go
// cmd/taskman/main.go — LE SEUL if du programme sur le mode
var repo task.Repository = storage.NewJSON(cfg.Path)
if cfg.DryRun {
	repo = storage.NewDryRun(repo, os.Stdout)
}
```

**Un seul `if`, au moment du câblage.** Tout le reste du programme ignore l'existence du mode
simulation. C'est le décorateur de la leçon 5 appliqué à un vrai besoin — et c'est le premier
usage réellement **architectural** d'une interface : elle ne sert pas à abstraire « au cas
où », elle sert à substituer un comportement.

L'anti-patron à éviter :
```go
if !dryRun {              // ← répété dans quinze fonctions
	if err := save(); err != nil { … }
}
```
Chaque nouvel appel à l'écriture devient une occasion d'oublier le test, et le bug ne se voit
qu'en production, quand la simulation écrit vraiment.

**Contrainte 4 — pas de duplication entre binaires :** les deux `main` importent le même
`internal/task` et le même `internal/storage`. Seule la couche `render` diffère.

## Défi

**a) Analyser un vrai projet**

Exemple avec `github.com/go-chi/chi` :

```
chi/
├── chi.go, mux.go, tree.go, context.go    ← API publique À LA RACINE
├── middleware/                             ← sous-package cohérent
├── _examples/                              ← préfixe _ : ignoré par l'outil go
└── go.mod                                  module github.com/go-chi/chi/v5
```

- **Pas de `cmd/`** : c'est une bibliothèque, il n'y a pas de binaire.
- **Pas de `pkg/`** : l'API publique est à la racine, comme dans la bibliothèque standard.
- **Pas d'`internal/`** — un choix discutable : tout est exporté, donc tout est un engagement
  de compatibilité. C'est aussi ce qui a rendu la migration `/v5` nécessaire.
- **Découpage par capacité**, pas par couche : `middleware` regroupe ce qui décore, le reste
  est le routeur.
- **Le `/v5`** dans le chemin de module : cinq versions majeures, cinq ruptures assumées.

**Une décision qu'on aurait prise autrement :** l'absence d'`internal/`. Les structures de
l'arbre de routage (`tree.go`) sont un détail d'implémentation exporté, ce qui interdit de les
changer sans version majeure. Les mettre sous `internal/` aurait donné plus de liberté sans rien
retirer aux utilisateurs.

*(Le même exercice sur `cobra` ou `bubbletea` donne des réponses différentes — c'est le but :
il n'existe pas de structure unique.)*

**b) Détecter la sur-structuration**

```go
type PkgStats struct {
	Path       string
	Files      int
	Lines      int
	Exported   int
	ImportedBy int
}
```

Un critère chiffré défendable :

> Un package est **sur-structuré** s'il a moins de 100 lignes **et** est importé par un seul
> autre package **et** exporte plus d'un tiers de ses identifiants.

Le raisonnement derrière chaque terme :
- **moins de 100 lignes** : le package ne porte pas assez de logique pour mériter une
  frontière ;
- **importé par un seul** : la frontière ne sert à personne — c'est du découpage décoratif ;
- **plus d'un tiers exporté** : la frontière a *forcé* à exporter, elle a donc dégradé
  l'encapsulation au lieu de l'améliorer.

Les trois conditions doivent être réunies : un petit package importé par dix autres est un
**bon** package (une abstraction largement réutilisée), et un gros package n'exportant presque
rien est excellent.

Ce qui compte dans cet exercice n'est pas le seuil exact — il est discutable — mais le fait de
**rendre explicite une intuition**. C'est précisément ce qu'on attend d'un développeur senior
en revue d'architecture : « ça ne me plaît pas » n'est pas un argument, « ce package coûte trois
frontières pour un seul appelant » en est un.

**c) Le débat `pkg/`**

**Pour.** `pkg/` complète `internal/` en rendant l'intention symétrique et lisible : d'un coup
d'œil, on sait ce qui est destiné à l'extérieur et ce qui ne l'est pas. Sur un gros dépôt
partagé par plusieurs équipes, cette signalisation vaut mieux qu'une convention orale. Elle
évite qu'un composant devienne une API publique par accident, simplement parce que quelqu'un
l'a importé. Enfin, elle offre un endroit évident où déposer le code qu'on assume de
maintenir — et le fait que le compilateur ne l'impose pas n'annule pas sa valeur documentaire,
pas plus que les commentaires ne sont inutiles parce qu'ils ne compilent pas.

**Contre.** `pkg/` n'a **aucun effet** : il n'empêche rien, ne garantit rien, et le compilateur
l'ignore. Il allonge tous les chemins d'import d'un segment qui n'apporte aucune information —
`monprojet/pkg/client` ne dit rien de plus que `monprojet/client`. En pratique, la plupart des
projets qui l'adoptent y mettent **tout** par mimétisme, ce qui lui retire jusqu'à sa valeur
documentaire. Et la bibliothèque standard, référence absolue du style Go, ne l'utilise pas :
`net/http` n'est pas `net/pkg/http`. La signalisation qu'il prétend apporter est mieux servie
par `internal/`, qui elle est réelle : ce qui n'est pas sous `internal/` est public, point.

**Position et critère.** Ne pas utiliser `pkg/` par défaut. Le critère qui pourrait faire
changer d'avis : un **monorepo multi-équipes** où la même arborescence contient des applications
et des bibliothèques partagées, et où la distinction doit sauter aux yeux sans lire les
`go.mod`. Hors de ce cas, `internal/` suffit — et la question « est-ce public ? » se règle en
regardant si le chemin contient `internal`.

*(Russ Cox et Dave Cheney se sont tous deux exprimés contre. L'équipe Go a publiquement pris
ses distances avec `golang-standards/project-layout`, qui popularise cette structure sans être
officiel.)*

## Réponses du quiz

1. **Non**, Go n'impose aucune structure. `golang-standards/project-layout` n'est **pas
   officiel** et l'équipe Go s'en est explicitement distanciée.
2. À partir de **deux** exécutables. Avec un seul, `main.go` à la racine est plus simple.
3. `internal/` est appliqué par le **compilateur** : l'import depuis l'extérieur est refusé.
   `pkg/` n'est qu'un nom de répertoire, sans aucun effet.
4. Il crée des **cycles** dès qu'une relation devient bidirectionnelle ; il force à **exporter**
   des identifiants qui auraient dû rester privés ; et il disperse un même changement métier sur
   plusieurs packages sans cohésion.
5. Parce que `os.Exit` **n'exécute aucun `defer`**. En le confinant à `main`, tous les `defer` de
   `run` s'exécutent. De plus, `run` retourne une erreur, donc elle est **testable** ; `main` ne
   l'est pas.
6. Pour la raison ci-dessus, et parce qu'un `os.Exit` dispersé rend le flux de sortie du
   programme impossible à suivre.
7. Pour rendre `run` **testable sans processus** : un `bytes.Buffer` remplace le terminal.
   C'est aussi ce qui permet de rediriger la sortie sans toucher au code.
8. **À côté du code**, dans le même répertoire : `user_test.go` près de `user.go`. Pas de
   répertoire `tests/`.
9. Hésiter plus de dix secondes sur l'endroit où placer une nouvelle fonction. Cela signale un
   découpage trop fin, ou fondé sur le mauvais critère.
10. Commencer **plat** — un fichier, puis plusieurs fichiers dans un seul package — et ne
    découper que lorsque la douleur se manifeste, selon la ligne de fracture qui s'est révélée
    d'elle-même. Un découpage tardif et juste vaut mieux qu'un découpage précoce et faux.

---

## 🎓 Fin du niveau 2

Une fois ces dix leçons digérées : demander l'**évaluation cumulative** des niveaux 1 et 2,
puis attaquer le [projet 1 — `wordstat`](../../projets/projet-01-cli-simple/).
