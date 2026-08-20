# Leçon 10 — Organisation d'un projet Go

> Dernière leçon du niveau 2. Elle synthétise les précédentes : packages, visibilité,
> interfaces, modules. Elle prépare le **projet 1**.

## Objectifs

1. Structurer un projet Go réel, du script de 200 lignes au service complet.
2. Comprendre ce que valent réellement `cmd/`, `internal/` et `pkg/`.
3. Savoir découper par **domaine** plutôt que par couche technique — et pourquoi.
4. Reconnaître la sur-structuration, aussi coûteuse que l'absence de structure.

## Explication

### Il n'y a pas de structure officielle

Contrairement à Rails, Django ou Spring, Go **n'impose aucune arborescence**. La bibliothèque
standard elle-même est un simple ensemble de répertoires plats.

Le dépôt `golang-standards/project-layout`, très étoilé, n'est **pas officiel** et l'équipe Go
s'en est explicitement distanciée. Il décrit ce que font certains grands projets, ce qui n'en
fait pas une recommandation pour un projet de 2 000 lignes.

La règle réelle est celle de la leçon 8 : **structurer quand la douleur apparaît**, pas avant.

### Quatre stades de croissance

**Stade 1 — un fichier.** Jusqu'à ~300 lignes.
```
outil/
├── go.mod
└── main.go
```
C'est parfaitement acceptable. Beaucoup d'outils en ligne de commande n'ont jamais besoin de
plus.

**Stade 2 — plusieurs fichiers, un seul package.** Jusqu'à ~1 000 lignes.
```
outil/
├── go.mod
├── main.go        analyse des arguments, orchestration, sortie
├── stats.go       le calcul
└── scan.go        la lecture
```
Le découpage est purement organisationnel : tout est dans `package main`, tout se voit. C'est
exactement ce que demande le **projet 1**.

**Stade 3 — plusieurs packages.** Quand une partie du code mérite son propre vocabulaire, ou
doit être testable indépendamment.
```
projet/
├── go.mod
├── main.go
└── internal/
    ├── catalog/
    └── format/
```

**Stade 4 — plusieurs binaires.**
```
projet/
├── cmd/
│   ├── server/main.go
│   └── migrate/main.go
└── internal/
```

`cmd/` n'a de sens qu'à partir de **deux** exécutables. Avec un seul, `main.go` à la racine est
plus simple et tout aussi correct.

### `internal/`, `pkg/` : ce qu'ils valent

**`internal/` est un mécanisme du compilateur** (leçon 8) : il empêche réellement l'import
depuis l'extérieur. C'est le seul des trois qui a une sémantique. **À utiliser massivement** :
tout ce qui n'est pas une API publique délibérée devrait y vivre. Sans lui, chaque type
exporté devient un engagement de compatibilité involontaire.

**`pkg/` n'est qu'une convention de nommage**, sans aucun effet. Il est très discuté :

- *Pour* : il signale « ceci est réutilisable par l'extérieur », en miroir d'`internal/`.
- *Contre* : il ajoute un niveau à tous les chemins d'import sans rien garantir, et beaucoup
  de projets y mettent tout par mimétisme, ce qui lui retire tout sens.

**Recommandation :** ne pas utiliser `pkg/` par défaut. Un projet applicatif met son code sous
`internal/`. Une bibliothèque met ses packages à la racine — comme le fait la bibliothèque
standard.

### Découper par domaine, pas par couche

C'est **la** décision structurante, et la plus contre-intuitive pour qui vient de Java ou de
Rails.

```
✘ Par couche technique                ✔ Par domaine
internal/                             internal/
├── models/                           ├── user/
│   ├── user.go                       │   ├── user.go        (le type)
│   ├── order.go                      │   ├── service.go     (la logique)
│   └── product.go                    │   └── store.go       (la persistance)
├── services/                         ├── order/
│   ├── user.go                       │   ├── order.go
│   └── order.go                      │   ├── service.go
└── repositories/                     │   └── store.go
    ├── user.go                       └── catalog/
    └── order.go
```

Pourquoi le second résiste mieux :

1. **Cohésion.** Tout ce qui concerne l'utilisateur est au même endroit. Une modification
   touche un répertoire, pas trois.
2. **Cycles.** Le découpage par couche crée des cycles dès que `models` a besoin d'un service.
   Le découpage par domaine dirige naturellement les dépendances.
3. **Encapsulation réelle.** `user` peut avoir des types non exportés utilisés par sa logique
   et son stockage — impossible quand ils sont dans trois packages différents, ce qui force à
   tout exporter.
4. **Suppression.** Retirer une fonctionnalité, c'est supprimer un répertoire.

L'honnêteté oblige à nuancer : sur un projet de quatre types et deux écrans, le découpage par
couche fonctionne très bien et se lit facilement. Le problème n'apparaît qu'à partir de
quelques milliers de lignes — mais il apparaît **toujours**, et la migration est douloureuse.

### `main` doit être mince

```go
func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, "erreur :", err)
		os.Exit(1)
	}
}

func run() error {
	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("configuration : %w", err)
	}
	// … câblage des dépendances …
	return srv.Start()
}
```

Ce patron `main` + `run() error` est un idiome répandu et vaut d'être adopté tout de suite.
Raisons :

- **`os.Exit` n'exécute aucun `defer`** (leçon 7). En le confinant à `main`, tous les `defer`
  de `run` s'exécutent normalement.
- `run()` retourne une erreur, donc elle est **testable** ; `main` ne l'est pas.
- La gestion du code de sortie est concentrée en un seul endroit.

Le rôle de `main` est le **câblage** : lire la configuration, construire les dépendances, les
brancher, démarrer. Aucune logique métier.

### Où vont les tests, la configuration, les données

- **Tests** : à côté du code, dans le même répertoire (`user_test.go` près de `user.go`).
  Pas de répertoire `tests/`. Détail au niveau 5.
- **Configuration** : lue depuis l'environnement, avec des valeurs par défaut dans le code.
  Un package `config` dédié, chargé une fois dans `run()`.
- **Fichiers de données** (migrations SQL, gabarits, ressources) : à côté du code qui les
  utilise, embarqués dans le binaire avec `//go:embed` (niveau 4). Un binaire Go autonome qui
  dépend de fichiers externes perd une grande partie de son intérêt.
- **Documentation** : `README.md` à la racine, commentaire `// Package x …` dans chaque
  package, éventuellement un `doc.go`.

### Trois structures qui marchent

**Outil en ligne de commande**
```
wordstat/
├── go.mod
├── README.md
├── main.go
├── stats.go
└── stats_test.go
```

**Service HTTP de taille moyenne**
```
service/
├── go.mod
├── main.go
└── internal/
    ├── config/
    ├── http/            handlers, middleware, routage
    ├── user/            domaine : type, logique, stockage
    ├── order/
    └── storage/         accès base partagé (pool, migrations)
```

**Bibliothèque**
```
slugify/
├── go.mod
├── README.md
├── LICENSE
├── slugify.go           API publique à la racine
├── slugify_test.go
└── internal/
    └── tables/          détails d'implémentation, non importables
```

### Les pièges de la sur-structuration

Un projet de 500 lignes réparti sur douze packages coûte plus qu'il ne rapporte :

- chaque frontière de package force à **exporter** des identifiants qui auraient dû rester
  privés ;
- les cycles apparaissent et il faut inventer des interfaces pour les casser ;
- le moindre ajout demande de choisir un package, et la réponse n'est jamais évidente ;
- la navigation devient plus lente qu'avec un seul fichier de 500 lignes.

**Signal fiable :** si l'on hésite plus de dix secondes sur l'endroit où mettre une nouvelle
fonction, le découpage est mauvais — trop fin, ou fondé sur le mauvais critère.

La bonne trajectoire est **un seul package jusqu'à ce que ce soit douloureux**, puis découper
selon la ligne de fracture qui s'est révélée d'elle-même. Un découpage tardif et juste vaut
mieux qu'un découpage précoce et faux.

## Exemple

```
biblio/
├── go.mod                       module github.com/willbrid/biblio
├── README.md
├── main.go                      ~40 lignes : main + run
└── internal/
    ├── config/
    │   └── config.go            lecture de l'environnement, valeurs par défaut
    ├── catalog/                 DOMAINE : livres
    │   ├── catalog.go           le type et ses opérations
    │   ├── isbn.go              validation
    │   └── store.go             persistance en mémoire
    └── cli/
        ├── cli.go               analyse des sous-commandes
        └── render.go            affichage tabulaire
```

```go
// main.go — mince, testable, sans logique.
package main

import (
	"fmt"
	"os"

	"github.com/willbrid/biblio/internal/catalog"
	"github.com/willbrid/biblio/internal/cli"
	"github.com/willbrid/biblio/internal/config"
)

func main() {
	if err := run(os.Args[1:], os.Stdout, os.Stderr); err != nil {
		fmt.Fprintln(os.Stderr, "erreur :", err)
		os.Exit(1) // seul endroit du programme où os.Exit apparaît
	}
}

// run contient tout le câblage. Elle prend ses entrées/sorties en paramètres,
// ce qui la rend testable sans toucher au système (niveau 5).
func run(args []string, stdout, stderr *os.File) error {
	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("configuration : %w", err)
	}

	cat := catalog.New(catalog.WithMaxBooks(cfg.MaxBooks))

	app := cli.New(cat, stdout, stderr)
	return app.Run(args)
}
```

```go
// internal/catalog/catalog.go
// Package catalog gère le référentiel de livres et ses règles métier.
// Une phrase, sans « et » : le découpage est probablement bon.
package catalog

// Le domaine ne connaît NI l'affichage, NI la ligne de commande, NI la config.
// Il ne dépend que de la bibliothèque standard. C'est ce qui le rend testable
// et réutilisable — et c'est le point de départ du niveau 10 (architecture).
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `main` + `run() error` | `os.Exit` confiné à `main` : tous les `defer` de `run` s'exécutent. |
| `run(args, stdout, stderr)` | Les entrées/sorties sont des **paramètres**, pas des globales. La fonction devient testable sans processus. |
| Un seul `os.Exit` | Facile à auditer : un `grep os.Exit` doit retourner une seule ligne. |
| `internal/` partout | Rien n'est une API publique : le projet garde toute liberté d'évolution. |
| `catalog` sans dépendance | Le domaine ne connaît ni la CLI, ni la configuration. Les dépendances vont **vers** lui, jamais l'inverse. |
| `cli` dépend de `catalog` | La couche externe dépend du cœur, pas le contraire. C'est le principe qui sera formalisé au niveau 10. |
| `WithMaxBooks(...)` | Options fonctionnelles (niveau 1, leçon 11) : la configuration entre dans le domaine sans qu'il connaisse le package `config`. |

## Erreurs fréquentes

1. **Copier `project-layout` sur un projet de 300 lignes** : douze répertoires vides.
2. **`cmd/` avec un seul binaire** : un niveau de chemin pour rien.
3. **Ne rien mettre sous `internal/`** : tout devient une API publique involontaire.
4. **Découper par couche technique** : cycles et absence de cohésion à mesure que le projet grossit.
5. **Un package par type** : friction maximale, exports forcés.
6. **`main()` de 300 lignes** avec la logique métier dedans.
7. **`os.Exit` dispersé** : les `defer` ne s'exécutent pas, et le flux devient impossible à suivre.
8. **Variables globales de configuration** : rend tout le code dépendant d'un état implicite et intestable.
9. **Répertoire `tests/`** : ce n'est pas la convention Go.
10. **Structurer avant d'avoir mal.**

## Bonnes pratiques Go

- Commencer **plat**, découper quand la douleur se manifeste.
- `internal/` par défaut ; `pkg/` seulement avec une raison explicite ; `cmd/` à partir de deux binaires.
- Découper par **domaine**, pas par couche.
- `main` mince, patron `run() error`, un seul `os.Exit`.
- Entrées/sorties et dépendances passées en **paramètres**, jamais en globales.
- Le cœur métier ne dépend de rien d'extérieur.
- Un commentaire de package d'une phrase — si elle contient « et », interroger le découpage.
- Ressources embarquées dans le binaire plutôt que fichiers externes.

## Ce que je dois retenir

- Go **n'impose aucune structure** ; `project-layout` n'est pas officiel.
- Quatre stades : un fichier → plusieurs fichiers → plusieurs packages → plusieurs binaires.
- **`internal/` est réel** (contrôlé par le compilateur), **`pkg/` n'est qu'un nom**.
- Découper **par domaine** ; le découpage par couche crée cycles et exports inutiles.
- `main` mince + `run() error` : testable, et les `defer` s'exécutent.
- La sur-structuration coûte autant que l'absence de structure.

---

## 🎓 Fin du niveau 2

Prochaines étapes :
1. **Évaluation cumulative** des niveaux 1 et 2 (à demander au professeur).
2. **[Projet 1 — `wordstat`](../../projets/projet-01-cli-simple/)**.
3. Puis le [niveau 3 — Go moderne et génériques](../../niveau-03-go-moderne-generiques/).

➡️ [Exercices](exercices.md)
