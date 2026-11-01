# Leçon 8 — Outillage qualité

> Les tests vérifient ce que le code **fait**. Les analyseurs statiques lisent ce que le code
> **est**, sans l'exécuter, et trouvent des bugs que personne n'a pensé à tester : un `Printf`
> aux mauvais arguments, une erreur ignorée, une fonction dépréciée, une dépendance vulnérable.
> Cette leçon présente la chaîne d'outils qu'attend un projet open source Go — et un piège très
> concret qu'on rencontre au passage d'une version de Go à la suivante.

## Objectifs

1. Connaître `go vet`, `go fix`, `staticcheck`, `golangci-lint` et `govulncheck`, et ce que chacun apporte.
2. Moderniser du code automatiquement avec `go fix` (Go 1.26+).
3. Configurer un linter sans se noyer dans les avertissements.
4. Intégrer ces outils au flux de travail (avant commit, en CI).

## Explication

### La chaîne, du plus fondamental au plus large

| Outil | Livré avec Go ? | Ce qu'il trouve | Faux positifs |
|---|---|---|---|
| `gofmt` / `goimports` | oui / module `x/tools` | formatage, imports | aucun |
| `go vet` | oui | erreurs **certaines** : `Printf` mal formé, copie de verrou, code inatteignable, `struct` tags invalides… | quasiment aucun |
| `go fix` | oui | code **datable** à moderniser, avec correction automatique | aucun (les corrections sont sûres) |
| `staticcheck` | non | bugs probables, API dépréciées, simplifications | très peu |
| `golangci-lint` | non | **agrégateur** de dizaines de linters (dont `staticcheck`, `errcheck`, `revive`…) | selon la configuration |
| `govulncheck` | non (module officiel `x/vuln`) | dépendances et appels **vulnérables** connus | très peu : il vérifie que le code vulnérable est **appelé** |

### `go vet`

`go vet ./...` lance une série d'analyseurs choisis pour ne signaler **que** des problèmes réels.
`go test` en exécute automatiquement un sous-ensemble (dont `printf` et, depuis Go 1.27,
`stdversion`) : un test ne passe pas si `vet` trouve une faute grave.

```
$ go vet ./...
app.go:33:14: fmt.Printf format %d has arg cfg.Name of wrong type string
```

Règle de projet : **`go vet` doit être silencieux**. Pas d'exception, pas de « on verra ».

### `go fix` : moderniser automatiquement

Historiquement, `go fix` réécrivait du code pour suivre des changements d'API d'avant Go 1. Depuis
Go 1.26, il a été entièrement refondu autour des **modernisateurs** : des analyseurs qui repèrent
des constructions anciennes et proposent la forme moderne, avec une correction **sûre** (même
comportement).

```bash
go fix -diff ./...     # montre les changements sans les appliquer
go fix ./...           # les applique
go tool fix help       # liste des analyseurs
```

Quelques modernisateurs (liste complète avec `go tool fix help`) :

| Analyseur | Avant | Après |
|---|---|---|
| `any` | `interface{}` | `any` |
| `rangeint` | `for i := 0; i < n; i++` | `for i := range n` |
| `slicescontains` | boucle de recherche | `slices.Contains` |
| `minmax` | `if a > b { m = a } else { m = b }` | `max(a, b)` |
| `stringscut` | `strings.Index` + découpage | `strings.Cut` |
| `stringsseq` | `range strings.Split(…)` | `range strings.SplitSeq(…)` |
| `forvar` | `tt := tt` dans une boucle | supprimé (N5 L2) |
| `waitgroupgo` | `wg.Add(1); go func(){ defer wg.Done() … }()` | `wg.Go(…)` (N6) |
| `omitzero` | `omitempty` sur un champ struct | `omitzero` (N4 L6) |
| `inline` | appel d'une fonction marquée `//go:fix inline` | son corps |

Ce dernier est une arme pour les **auteurs de bibliothèques** : une fonction dépréciée annotée
`//go:fix inline` sera remplacée automatiquement, chez tous les utilisateurs qui lancent `go fix`,
par son implémentation (qui appelle la nouvelle API). La migration d'une API devient un outil.

Les corrections respectent la ligne `go` du `go.mod` : `go fix` ne propose pas `slices.Contains`
à un module déclaré en `go 1.20`.

### `staticcheck`

L'analyseur indépendant le plus respecté de l'écosystème. Ses vérifications ont des identifiants :
`SA…` (bugs probables, par exemple `SA1019` : usage d'une API dépréciée comme `strings.Title`),
`S…` (simplifications), `ST…` (style), `QF…` (corrections rapides). Il s'installe avec :

```bash
go install honnef.co/go/tools/cmd/staticcheck@latest
```

### Le piège de la version

Un analyseur est **compilé avec une version de Go** et charge les informations de type produites
par la chaîne installée. Quand on passe à une nouvelle version de Go, un analyseur ancien peut
devenir **incapable de lire le code**. Sur la machine de rédaction, après la mise à jour vers Go
1.27, les deux outils installés précédemment échouaient :

```
$ staticcheck ./...
-: internal error in importing "internal/cpu" (cannot decode "internal/cpu", export data version 4
   is greater than maximum supported version 2); please report an issue (compile)

$ golangci-lint run ./...
Error: can't load config: the Go language version (go1.26) used to build golangci-lint
   is lower than the targeted Go version (1.27)
```

`go version -m $(which staticcheck)` montre avec quelle version de Go un binaire a été construit.
La correction : **réinstaller les outils** après chaque mise à jour de Go (et attendre, au besoin,
qu'ils publient une version compatible). En CI, on fixe la version de chaque outil pour éviter les
surprises — on y revient au niveau 12.

### `golangci-lint`

Un exécutable qui lance en parallèle des dizaines de linters, avec une configuration unique
(`.golangci.yml`) et un cache. C'est l'outil le plus répandu dans les projets open source. Le
danger est d'activer **tous** les linters : des centaines d'avertissements de style, contradictoires
entre eux, que l'équipe finit par ignorer — y compris les vrais.

Une configuration raisonnable de départ :

```yaml
# .golangci.yml
version: "2"
linters:
  default: standard        # govet, errcheck, staticcheck, unused, ineffassign
  enable:
    - revive               # style, successeur de golint
    - gosec                # problèmes de sécurité courants
    - errorlint            # comparaisons d'erreurs sans errors.Is / errors.As
    - bodyclose            # corps de réponse HTTP non fermés (niveau 7)
```

On **ajoute** un linter quand un type de bug est revenu deux fois, pas avant.

### `govulncheck`

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

Il compare les dépendances du module (et la bibliothèque standard de la version de Go utilisée) à
la base officielle des vulnérabilités Go, **puis** analyse le graphe d'appels : il ne signale une
vulnérabilité que si le code vulnérable est **effectivement appelé**. C'est ce qui le rend
utilisable au quotidien, là où un scanner de dépendances classique noie l'équipe sous des alertes
sans rapport. À lancer en CI, et avant chaque version. Le niveau 12 (leçon 8) y revient.

### Ignorer un avertissement

Tous les outils permettent de faire taire une alerte précise :

```go
//nolint:errcheck // l'erreur de Close en lecture n'apporte rien (N4 L7)
defer f.Close()
```

La règle : **toujours avec une justification**, et sur la ligne concernée, jamais sur tout un
fichier. Un `//nolint` sans raison est refusé en revue.

### Où les lancer

| Moment | Outils |
|---|---|
| à chaque sauvegarde (éditeur, `gopls`) | `gofmt`, `go vet`, diagnostics `staticcheck` |
| avant chaque commit | `go vet ./...`, `go test -race ./...` |
| en CI, sur chaque *pull request* | tout : `gofmt -l`, `vet`, `golangci-lint`, `go test -race`, `govulncheck` |
| ponctuellement | `go fix -diff` (après une mise à jour de Go), fuzzing long |

## Exemple

Un fichier qui « marche » sur le cas nominal :

```go
package main

import (
	"fmt"
	"os"
	"sort"
	"strings"
)

type Config struct{ Name string }

func load(path string) (*Config, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	data := make([]byte, 100)
	f.Read(data)
	return &Config{Name: strings.Title(string(data))}, nil
}

func contains(xs []string, v string) bool {
	for _, x := range xs {
		if x == v {
			return true
		}
	}
	return false
}

func main() {
	cfg, err := load("app.conf")
	fmt.Printf("%d\n", cfg.Name)
	if err != nil {
		fmt.Println("erreur")
	}
	keys := []string{"b", "a"}
	sort.Strings(keys)
	for i := 0; i < len(keys); i++ {
		fmt.Println(keys[i], contains(keys, "a"))
	}
	var m interface{} = cfg
	_ = m
}
```

`go vet ./...` :

```
app.go:33:14: fmt.Printf format %d has arg cfg.Name of wrong type string
```

`go fix -diff ./...` (extrait) :

```diff
 func contains(xs []string, v string) bool {
-	for _, x := range xs {
-		if x == v {
-			return true
-		}
-	}
-	return false
+	return slices.Contains(xs, v)
 }
…
-	for i := 0; i < len(keys); i++ {
+	for i := range keys {
…
-	var m interface{} = cfg
+	var m any = cfg
```

## Explication du code

| Ligne | Trouvé par | Problème |
|---|---|---|
| `fmt.Printf("%d\n", cfg.Name)` | **`go vet`** | verbe `%d` pour une chaîne |
| `cfg.Name` **avant** `if err != nil` | **aucun outil de cette leçon** ; un test oui | si le fichier est absent, `cfg` est nil : panique. L'erreur doit être testée **avant** d'utiliser le résultat (N1 L10). |
| `f.Read(data)` | `errcheck` (dans `golangci-lint`) | résultat et erreur ignorés ; et un seul `Read` ne lit pas forcément tout (N4 L5) |
| `os.Open` sans `Close` | une revue humaine | fuite de descripteur |
| `strings.Title` | `staticcheck` (`SA1019`) | fonction dépréciée (N4 L1) |
| `make([]byte, 100)` puis `string(data)` | une revue humaine | les octets nuls restants finissent dans le nom |
| boucle de `contains` | **`go fix`** (`slicescontains`) | à remplacer par `slices.Contains` |
| `for i := 0; i < len(keys); i++` | **`go fix`** (`rangeint`) | forme moderne `for i := range keys` |
| `interface{}` | **`go fix`** (`any`) | |

Le tableau dit l'essentiel : les outils trouvent **une partie** des problèmes, chacun la sienne.
Le plus grave (la panique sur `cfg` nil) n'est trouvé que par un **test** du cas d'erreur. Les
outils ne remplacent ni les tests ni la revue ; ils les libèrent de ce qui peut être automatisé.

## Erreurs fréquentes

1. **Ne lancer que `go build`** : il compile, donc « ça marche ».
2. **Activer tous les linters** de `golangci-lint` et ignorer ensuite la masse d'avertissements.
3. **`//nolint` sans justification**, ou sur un fichier entier.
4. **Oublier de réinstaller les outils** après une mise à jour de Go.
5. **Appliquer `go fix` sans relire le diff** dans un gros projet (les corrections sont sûres,
   mais un commit de 2 000 lignes mélangé à une fonctionnalité est irrelisible).
6. **Ne pas fixer les versions des outils en CI** : une nouvelle version d'un linter casse la CI
   un lundi matin.
7. **Prendre un outil vert pour une preuve de qualité.**

## Bonnes pratiques Go

- `go vet` silencieux, toujours.
- `go fix -diff` après chaque mise à jour de Go, appliqué dans un commit **dédié**.
- `golangci-lint` avec une configuration minimale, enrichie au besoin.
- `govulncheck` en CI et avant chaque version.
- Réinstaller (ou mettre à jour) les outils à chaque changement de version de Go ; versions fixées en CI.
- `//nolint:linter // raison`, sur la ligne, et rien d'autre.

## Ce que je dois retenir

- `go vet` : erreurs certaines, lancé en partie par `go test`.
- `go fix` (Go 1.26+) : modernisateurs à correction sûre ; `-diff` pour voir ; `//go:fix inline` pour migrer une API.
- `staticcheck` : bugs probables et API dépréciées ; `golangci-lint` : agrégateur configurable.
- `govulncheck` : vulnérabilités **réellement atteintes** par le code.
- Un analyseur compilé avec une version de Go plus ancienne peut ne plus fonctionner : réinstaller.
- Les outils complètent les tests et la revue ; ils ne les remplacent pas.

---

➡️ [Exercices](exercices.md)
