# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir réellement essayé. Une solution lue n'apprend presque rien ;
> une solution comparée à sa propre tentative apprend beaucoup.

## E1 — Environnement

```bash
$ go version
go version go1.27.0 linux/amd64
```

| Variable | Rôle |
|---|---|
| `GOROOT` | Où Go lui-même est installé (`/usr/local/go`) : compilateur, outils, bibliothèque standard. **Ne jamais la définir à la main** : Go la déduit. |
| `GOPATH` | Espace de travail historique (`~/go`). Depuis les modules, il ne sert plus qu'à héberger `bin/` (binaires installés par `go install`) et `pkg/mod/` (cache). |
| `GOMODCACHE` | Cache des modules téléchargés (`$GOPATH/pkg/mod`). Partagé par tous les projets ; `go clean -modcache` le vide. |

## E2 — Introspection

```go
package main

import (
	"fmt"
	"os"
	"runtime"
)

func main() {
	fmt.Printf("programme : %s\n", os.Args[0])
	fmt.Printf("Go        : %s\n", runtime.Version())
	fmt.Printf("plateforme: %s/%s\n", runtime.GOOS, runtime.GOARCH)
	fmt.Printf("CPU       : %d cœurs\n", runtime.NumCPU())
}
```

`runtime.Version()` renvoie la version **du compilateur qui a produit le binaire**, pas
celle installée aujourd'hui. C'est utile pour diagnostiquer un déploiement.

## E3 — Arguments

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	for i, arg := range os.Args[1:] {
		fmt.Printf("%d: %s\n", i+1, arg)
	}
}
```

Points de correction fréquents :
- `os.Args[1:]` et non `os.Args` — sinon le chemin du programme est listé.
- `i+1` parce que `range` commence à 0.
- Un `for` classique avec `len()` fonctionne aussi, mais `range` est idiomatique ici.

## E4 — Les messages du compilateur

```
./main.go:4:2: "strings" imported and not used
./main.go:7:2: declared and not used: x
./main.go:6:1: syntax error: unexpected semicolon or newline before {
```

Le troisième est le plus instructif : Go a inséré un point-virgule automatique après
`func main()`. Le message parle d'un « semicolon » qu'on n'a jamais tapé — comprendre
cette insertion automatique explique la moitié des erreurs de syntaxe en Go.

## E5 — Le binaire

Environ **1,2 à 2 Mo** pour un « hello world ». Le binaire embarque :
le runtime Go (ordonnanceur de goroutines, ramasse-miettes, allocateur), les parties de la
bibliothèque standard utilisées, et les tables de types nécessaires à la réflexion et aux
traces de pile.

C'est le prix du déploiement sans dépendance. À comparer : un `.class` Java fait quelques
kilo-octets mais exige une JVM de 200 Mo. `go build -ldflags="-s -w"` retire les tables de
symboles et de debug et fait tomber la taille d'environ 30 % — utile en conteneur, au prix
de traces de pile moins lisibles.

## Exercice intermédiaire — `tempconv`

```go
package main

import (
	"fmt"
	"os"
	"strconv"
	"strings"
)

// CToF convertit des degrés Celsius en Fahrenheit.
// Fonction pure : aucun affichage, aucun accès au système. Donc testable.
func CToF(c float64) float64 { return c*9/5 + 32 }

// FToC convertit des degrés Fahrenheit en Celsius.
func FToC(f float64) float64 { return (f - 32) * 5 / 9 }

func main() {
	args := os.Args[1:]
	if len(args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: tempconv <valeur> <C|F>")
		os.Exit(1)
	}

	value, err := strconv.ParseFloat(args[0], 64)
	if err != nil {
		fmt.Fprintf(os.Stderr, "valeur invalide %q : %v\n", args[0], err)
		os.Exit(1)
	}

	switch strings.ToUpper(args[1]) {
	case "C":
		fmt.Printf("%.1f°C = %.1f°F\n", value, CToF(value))
	case "F":
		fmt.Printf("%.1f°F = %.1f°C\n", value, FToC(value))
	default:
		fmt.Fprintf(os.Stderr, "unité inconnue %q (attendu C ou F)\n", args[1])
		os.Exit(1)
	}
}
```

**Pourquoi `os.Stderr` et le code de sortie 1 ?** Parce qu'un shell compose des programmes :
`tempconv 100 C > resultat.txt` ne doit écrire dans le fichier que le résultat, pas le
message d'erreur. Et `tempconv … && suite` ne doit enchaîner que sur un succès. C'est le
contrat Unix, et le respecter est un critère « prêt pour la production ».

**Pourquoi `CToF` séparée ?** Parce qu'au niveau 3 on écrira `TestCToF` sans avoir à
simuler une ligne de commande ni capturer une sortie. Une fonction pure se teste en une ligne.

## Défi

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
	"strconv"
	"strings"
)

func CToF(c float64) float64 { return c*9/5 + 32 }
func FToC(f float64) float64 { return (f - 32) * 5 / 9 }

// convertLine traite une ligne « valeur unité » et retourne la ligne de sortie.
func convertLine(line string) (string, error) {
	fields := strings.Fields(line)
	if len(fields) != 2 {
		return "", fmt.Errorf("2 champs attendus, %d trouvés", len(fields))
	}
	v, err := strconv.ParseFloat(fields[0], 64)
	if err != nil {
		return "", fmt.Errorf("valeur %q : %w", fields[0], err)
	}
	switch strings.ToUpper(fields[1]) {
	case "C":
		return fmt.Sprintf("%.1f°C = %.1f°F", v, CToF(v)), nil
	case "F":
		return fmt.Sprintf("%.1f°F = %.1f°C", v, FToC(v)), nil
	default:
		return "", fmt.Errorf("unité inconnue %q", fields[1])
	}
}

// process lit r ligne par ligne et écrit dans w. Retourne le nombre d'erreurs.
func process(r io.Reader, w, errw io.Writer) int {
	sc := bufio.NewScanner(r)
	sc.Buffer(make([]byte, 0, 64*1024), 10*1024*1024) // lignes jusqu'à 10 Mo
	failures, n := 0, 0

	for sc.Scan() {
		n++
		line := strings.TrimSpace(sc.Text())
		if line == "" {
			continue
		}
		out, err := convertLine(line)
		if err != nil {
			fmt.Fprintf(errw, "ligne %d : %v\n", n, err)
			failures++
			continue
		}
		fmt.Fprintln(w, out)
	}
	if err := sc.Err(); err != nil { // NE JAMAIS oublier ce test
		fmt.Fprintf(errw, "lecture : %v\n", err)
		failures++
	}
	return failures
}

func main() {
	var input io.Reader = os.Stdin
	if args := os.Args[1:]; len(args) > 0 {
		input = strings.NewReader(strings.Join(args, " "))
	}
	if process(input, os.Stdout, os.Stderr) > 0 {
		os.Exit(1)
	}
}
```

**Les trois points qui font la différence :**

1. **`bufio.Scanner` lit ligne par ligne** : la mémoire utilisée est celle d'une ligne, pas
   du flux. `io.ReadAll` aurait chargé les 10 millions de lignes d'un coup.
2. **`sc.Buffer(…)`** : par défaut `Scanner` refuse les lignes de plus de 64 Ko et s'arrête
   avec `bufio.Scanner: token too long`. C'est un bug réel en production sur des logs JSON
   d'une seule ligne. Le connaître valait le défi à lui seul.
3. **`sc.Err()`** : `Scan()` retourne `false` aussi bien à la fin normale du flux que sur
   une erreur d'entrée/sortie. Sans ce test, une lecture tronquée passe pour un succès.

`process` prend un `io.Reader` et deux `io.Writer` : elle est testable sans fichier ni
terminal. Ce choix de signature est le vrai enseignement de la leçon — il annonce le projet 1.

## Réponses du quiz

1. **Faux.** Tous les fichiers d'un répertoire doivent déclarer le même package. Le package
   *est* le répertoire. (Seule exception : `package foo_test` pour les tests externes,
   niveau 3.)
2. `go run .` compile dans un répertoire temporaire et exécute sans laisser de binaire ;
   `go build` produit un exécutable persistant.
3. Parce qu'en Go la **casse de la première lettre** définit la visibilité : majuscule =
   exporté hors du package, minuscule = interne. Pas de mot-clé `public`/`private`.
4. **Erreur de compilation**, bloquante. Choix volontaire : les imports morts alourdissent
   les temps de compilation et trompent le lecteur. Go préfère un refus net à un
   avertissement qu'on ignore.
5. Elle déclare la **version du langage** demandée par le module : elle active les
   fonctionnalités correspondantes et fixe la sémantique (ex. la variable de boucle de
   Go 1.22). Ce n'est **pas** la version du compilateur utilisé, ni une version minimale
   d'installation.
6. `Printf` **écrit** sur la sortie standard ; `Sprintf` **retourne** la chaîne formatée.
7. Oui. Exemple : `fmt.Printf("%d", "texte")` compile mais est absurde ; `go vet` le
   signale. Autres cas : `append` dont le résultat est jeté, mutex copié, balises de struct
   malformées.
8. Le chemin du programme tel qu'invoqué. Les vrais arguments commencent à `os.Args[1]`.
