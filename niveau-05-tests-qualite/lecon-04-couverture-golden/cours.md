# Leçon 4 — Couverture et golden files

> Deux outils pour deux questions différentes. La **couverture** répond à « quelles lignes mes
> tests n'exécutent-ils jamais ? » — pas à « mes tests sont-ils bons ? ». Les **golden files**
> répondent à « comment tester une sortie de 200 lignes sans l'écrire dans le code du test ? ».

## Objectifs

1. Mesurer la couverture, la lire ligne par ligne, et savoir ce qu'elle **ne** dit **pas**.
2. Écrire des tests à fichiers de référence (*golden files*) avec un drapeau `-update`.
3. Organiser les données de test dans `testdata/`.

## Explication

### Mesurer la couverture

```bash
go test -cover ./...                          # pourcentage par paquet
go test -coverprofile=cover.out ./...         # profil détaillé
go tool cover -func=cover.out                 # pourcentage par fonction
go tool cover -html=cover.out                 # ouvre le code coloré dans le navigateur
```

La vue HTML colore chaque ligne : **vert** exécutée par au moins un test, **rouge** jamais
exécutée, gris non comptée (déclarations). C'est là que l'outil est utile : le rouge montre les
**branches oubliées** — presque toujours des chemins d'erreur.

`-covermode=count` compte combien de fois chaque ligne est exécutée (utile pour repérer le code
« chaud ») ; `-covermode=atomic` est requis avec `-race`. `-coverpkg=./...` mesure la couverture
de **tous** les paquets par les tests de chacun (sinon un paquet n'est couvert que par ses
propres tests).

### Ce que la couverture ne dit pas

```go
func Div(a, b int) int { return a / b }

func TestDiv(t *testing.T) {
	Div(6, 3)          // 100 % de couverture, et rien n'est vérifié
}
```

- **100 % ≠ correct** : une ligne exécutée n'est pas une ligne **vérifiée**. Le test ci-dessus
  n'a aucune assertion ; il ne trouve pas non plus la panique sur `Div(1, 0)`.
- La couverture est mesurée par **ligne** (par bloc, précisément), pas par **condition** :
  `if a && b` est « couvert » dès qu'une branche est prise, même si `b` n'est jamais évalué.
- Elle ne dit rien des **cas manquants** : un code qui ne gère pas les entrées vides est
  « couvert à 100 % » par des tests qui ne lui en donnent jamais.

Donc : la couverture est un outil pour **trouver ce qui n'est pas testé**, pas un objectif.
Viser 100 % produit des tests sans valeur écrits pour faire monter un chiffre. Une couverture
de 70–85 % sur le code métier, avec **tous** les chemins d'erreur importants verts, est un
meilleur signal qu'un 100 % obtenu à coups de tests sans assertions.

Une bonne pratique en revue : ouvrir la vue HTML sur le code **modifié** par la contribution, et
se demander pour chaque ligne rouge « est-ce un oubli, ou est-ce acceptable ? ».

### `testdata/`

Un répertoire nommé `testdata` est **ignoré** par l'outil `go` (il n'est jamais compilé, même
s'il contient des `.go`). C'est l'endroit conventionnel pour les fichiers d'entrée et de
référence des tests. `go test` s'exécute avec le **répertoire du paquet** comme répertoire
courant : `os.ReadFile("testdata/input.txt")` fonctionne directement.

### Les golden files

Pour une fonction qui produit une sortie volumineuse (rapport, rendu de gabarit, code généré,
réponse JSON formatée), écrire la sortie attendue **dans** le test est pénible et illisible. On
la met dans un fichier de référence :

```
testdata/
├── report-vide.input.json
├── report-vide.golden
├── report-trois-taches.input.json
└── report-trois-taches.golden
```

Le test lit l'entrée, produit la sortie, et la compare au fichier `.golden`. Quand la sortie
change **volontairement**, on régénère les fichiers avec un drapeau :

```go
var update = flag.Bool("update", false, "réécrire les fichiers golden")

func TestReport(t *testing.T) {
	got := Report(input)
	golden := filepath.Join("testdata", t.Name()+".golden")
	if *update {
		os.WriteFile(golden, got, 0o644)
	}
	want, err := os.ReadFile(golden)
	…
	if !bytes.Equal(got, want) { t.Errorf(…) }
}
```

```bash
go test ./report -update     # régénère
git diff testdata/           # ← l'étape essentielle : RELIRE le changement
```

Le drapeau `-update` est une variable de paquet du **test** (déclarée dans le `_test.go`) :
`go test` transmet au binaire de test les drapeaux qu'il ne connaît pas.

L'intérêt majeur : le **diff git** devient la revue du changement de sortie. Une modification
qui altère 40 lignes d'un rapport apparaît comme telle dans la *pull request*.

Le risque majeur : lancer `-update` par réflexe dès qu'un test échoue, et **valider un bug**
sans l'avoir vu. Le diff doit toujours être relu.

### Rendre la sortie déterministe

Un golden file ne fonctionne que si la sortie est **identique** à chaque exécution. Les ennemis :

- l'heure courante → horloge injectée (leçon 3) ;
- l'ordre des maps → trier (N3 L4) ;
- les chemins absolus de la machine (`/home/alice/…`) → chemins relatifs ;
- les identifiants aléatoires → générateur à graine fixe (N4 L3) ;
- les fins de ligne Windows → `.gitattributes` avec `*.golden -text`.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : report/report.go
// Package report produit un rapport texte des tâches.
package report

import (
	"cmp"
	"fmt"
	"slices"
	"strings"
	"time"
)

type Task struct {
	Title string
	Done  bool
	Due   time.Time
}

// Render produit le rapport. now est injecté : la sortie est déterministe.
func Render(tasks []Task, now time.Time) string {
	if len(tasks) == 0 {
		return "Aucune tâche.\n"
	}
	sorted := slices.Clone(tasks)
	slices.SortFunc(sorted, func(a, b Task) int {
		return cmp.Or(cmp.Compare(a.Due.Unix(), b.Due.Unix()), cmp.Compare(a.Title, b.Title))
	})

	var b strings.Builder
	late := 0
	for _, t := range sorted {
		mark := " "
		switch {
		case t.Done:
			mark = "x"
		case t.Due.Before(now):
			mark = "!"
			late++
		}
		fmt.Fprintf(&b, "[%s] %-20s %s\n", mark, t.Title, t.Due.Format("02/01"))
	}
	fmt.Fprintf(&b, "--\n%d tâche(s), %d en retard\n", len(tasks), late)
	return b.String()
}
```

```go
// Fichier : report/report_test.go
package report

import (
	"flag"
	"os"
	"path/filepath"
	"testing"
	"time"
)

var update = flag.Bool("update", false, "réécrire les fichiers golden de testdata/")

var now = time.Date(2026, 11, 10, 9, 0, 0, 0, time.UTC)

func day(d int) time.Time { return time.Date(2026, 11, d, 0, 0, 0, 0, time.UTC) }

func TestRender(t *testing.T) {
	tests := []struct {
		name  string
		tasks []Task
	}{
		{"vide", nil},
		{"melange", []Task{
			{Title: "Rapport", Due: day(12)},
			{Title: "Facture", Due: day(3)},
			{Title: "Courses", Due: day(9), Done: true},
			{Title: "Appel", Due: day(12)},
		}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := Render(tt.tasks, now)
			golden := filepath.Join("testdata", tt.name+".golden")
			if *update {
				if err := os.MkdirAll("testdata", 0o755); err != nil {
					t.Fatal(err)
				}
				if err := os.WriteFile(golden, []byte(got), 0o644); err != nil {
					t.Fatal(err)
				}
			}
			want, err := os.ReadFile(golden)
			if err != nil {
				t.Fatalf("lecture de %s : %v (lancer avec -update pour le créer)", golden, err)
			}
			if got != string(want) {
				t.Errorf("Render ne correspond pas à %s\n--- obtenu ---\n%s--- attendu ---\n%s", golden, got, want)
			}
		})
	}
}
```

Après `go test ./report -update`, le fichier `testdata/melange.golden` contient :

```
[!] Facture              03/11
[x] Courses              09/11
[ ] Appel                12/11
[ ] Rapport              12/11
--
4 tâche(s), 1 en retard
```

Et la couverture :

```
$ go test -cover ./report
ok  	exemple/report	0.003s	coverage: 100.0% of statements
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `now` en paramètre | Sans lui, le marqueur `!` dépendrait du jour où l'on lance le test : golden file impossible. |
| `slices.Clone` puis tri | Ordre déterministe, **et** le slice de l'appelant n'est pas réordonné. |
| Tri sur `Due` **puis** `Title` | `Appel` et `Rapport` ont la même échéance : sans le second critère, leur ordre ne serait pas garanti. |
| `var update = flag.Bool(…)` | Déclaré dans le fichier de test ; `go test … -update` le transmet. |
| `MkdirAll("testdata")` | Au tout premier `-update`, le répertoire n'existe pas encore : sans cette ligne, la première version de ce test échouait sur `no such file or directory`. |
| Message « lancer avec -update pour le créer » | Sans fichier de référence, le test échoue en disant **quoi faire**. |
| `got` et `want` dans le message | Pour une sortie longue, un vrai diff (`go-cmp`) est plus lisible ; ici, les deux blocs suffisent. |
| 100 % de couverture | Toutes les branches du `switch` sont prises **par les données du cas « melange »**, choisies pour ça : une tâche finie, une en retard, deux à venir. |

## Erreurs fréquentes

1. **Viser 100 %** et écrire des tests sans assertions pour l'atteindre.
2. **Croire qu'une ligne verte est une ligne vérifiée.**
3. **`-update` par réflexe**, sans relire `git diff`.
4. **Sortie non déterministe** (heure, ordre de map) dans un golden file : test instable.
5. **Golden files immenses** que personne ne relit : les découper en cas ciblés.
6. **Chemins de la machine** dans les sorties de référence.
7. **Ne pas regarder la vue HTML** : le pourcentage seul ne dit pas **où** sont les trous.

## Bonnes pratiques Go

- `go test -coverprofile` + `go tool cover -html` sur le code qu'on vient d'écrire.
- Chaque ligne rouge : un oubli à tester, ou un choix à assumer.
- `testdata/` pour toutes les données de test.
- Golden files pour les sorties volumineuses, avec un drapeau `-update` et une relecture du diff.
- Sorties déterministes : horloge injectée, tris explicites, graines fixes.
- Un cas de test par comportement, pas un golden file géant.

## Ce que je dois retenir

- `-cover`, `-coverprofile`, `go tool cover -html`, `-coverpkg`.
- La couverture montre ce qui n'est **pas** exécuté ; elle ne prouve rien sur ce qui l'est.
- `testdata/` est ignoré par l'outil `go` ; les tests s'exécutent dans le répertoire du paquet.
- Golden file : comparer à un fichier ; `-update` pour régénérer ; **relire le diff**.
- Un golden file exige une sortie **déterministe**.

---

➡️ [Exercices](exercices.md)
