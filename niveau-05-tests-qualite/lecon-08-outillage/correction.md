# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — L'état de ta machine

Sur la machine de rédaction (passée à Go 1.27 en août 2026), `go version -m` montrait par exemple
`staticcheck: go1.26.5` et `golangci-lint … built with go1.26.0`, et les deux échouaient sur un
module `go 1.27` (messages recopiés dans le cours). Après réinstallation avec une chaîne 1.27
(`go install …@latest`), ils fonctionnent — à condition que la dernière version publiée supporte
Go 1.27 ; sinon il faut attendre la version suivante de l'outil. Les autres outils (`dlv`, `gopls`)
suivent la même règle : `gopls` notamment doit être mis à jour pour comprendre les méthodes
génériques de 1.27.

## E2 — `go vet`

```go
type T struct {
	mu sync.Mutex
	N  int `json:"n" xml:"n,attr`          // structtag : tag mal formé
}

func byValue(t T) {}                       // copylocks : passe un Mutex par valeur

func f() {
	return
	fmt.Println("jamais")                  // unreachable
}

func g() {
	fmt.Printf("%s\n", 42)                 // printf : verbe/type
	fmt.Sprintf("x")                        // unusedresult : résultat de Sprintf ignoré
	var t T
	byValue(t)                              // copylocks
}
```

Sortie mesurée :

```
a.go:13:16: byValue passes lock by value: v.T contains sync.Mutex
a.go:24:10: call of byValue copies lock value: v.T contains sync.Mutex
a.go:21:14: fmt.Printf format %s has arg 42 of wrong type int
a.go:10:2: struct field tag `json:"n" xml:"n,attr` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
a.go:17:2: unreachable code
a.go:22:2: result of fmt.Sprintf call not used
```

(Avec `func f() int { return 1; fmt.Println(…) }`, ce n'est même pas `vet` qui répond mais le
compilateur : `missing return` — une fonction à résultat doit se terminer par une instruction
terminale.)

Le plus difficile à voir en relecture est presque toujours **`copylocks`** : passer une struct par
valeur est banal, et rien ne signale qu'elle contient un `sync.Mutex` (niveau 6) — copier un
verrou copie son état, et les deux copies ne se protègent plus mutuellement. Le tag mal formé
(guillemet manquant) est le second : il compile, et le champ est simplement ignoré par `xml`.

## E3 — Moderniser

Sur un code écrit avant ce niveau, les modernisateurs les plus fréquents sont `rangeint`
(`for i := 0; i < n; i++`), `any`, `slicescontains`, `minmax` et `stringscut`. Le commit
**séparé** compte : un relecteur peut le valider en sachant qu'il ne contient que des
transformations mécaniques, et `git bisect` (niveau 12) reste utile. Les tests doivent passer à
l'identique : les corrections de `go fix` ne changent pas le comportement.

## E4 — `//go:fix inline`

`go fix -diff` propose de remplacer chaque `Connect(addr)` par `NewClient(addr, DefaultTimeout)`,
avec les imports nécessaires. Pour les utilisateurs d'une bibliothèque, une dépréciation cesse
d'être une corvée manuelle : une commande migre tout leur code. Pour l'auteur, c'est un moyen de
faire **réellement** disparaître une ancienne API, ce qui était jusqu'ici presque impossible dans
un écosystème qui promet la compatibilité (niveau 13, leçon 6).

## E5 — Le tableau de l'exemple

```go
func load(path string) (*Config, error) {
	data, err := os.ReadFile(path)          // lit tout, ferme le fichier, rend l'erreur
	if err != nil {
		return nil, fmt.Errorf("lecture de la configuration : %w", err)
	}
	return &Config{Name: strings.TrimSpace(string(data))}, nil
}

func main() {
	cfg, err := load("app.conf")
	if err != nil {                         // l'erreur AVANT le résultat
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Printf("%s\n", cfg.Name)
	…
}
```

`strings.Title` supprimé (il ne servait qu'à « embellir » une donnée qu'on ne devait pas
transformer) ; plus d'octets nuls ; fichier fermé ; `Read` partiel impossible.

Le test qui aurait trouvé la panique :

```go
func TestLoadMissing(t *testing.T) {
	_, err := load(filepath.Join(t.TempDir(), "absent.conf"))
	if !errors.Is(err, fs.ErrNotExist) {
		t.Fatalf("load(absent) : erreur = %v, attendu fs.ErrNotExist", err)
	}
}
```

… et surtout un test de `main` (ou de `run`, N5 L1) sur un fichier absent, qui aurait paniqué. Le
bug était dans l'**ordre** des instructions, invisible pour les analyseurs de cette leçon, évident
pour un test du chemin d'erreur.

## Exercice intermédiaire — La chaîne qualité du projet 2

```makefile
.PHONY: check fmt vet fix lint test vuln

check: fmt vet fix lint test vuln

fmt:
	@test -z "$$(gofmt -l .)" || (gofmt -l . ; echo "gofmt : fichiers à formater" ; exit 1)

vet:
	go vet ./...

fix:
	@test -z "$$(go fix -diff ./...)" || (echo "go fix : modernisations en attente (go fix -diff ./...)" ; exit 1)

lint:
	golangci-lint run

test:
	go test -race -shuffle=on ./...

vuln:
	govulncheck ./...
```

- `$$` dans un `Makefile` pour obtenir un `$` dans le shell.
- L'ordre va du plus rapide au plus lent : on échoue vite sur les problèmes triviaux.
- Typiquement, `golangci-lint` (à froid, sans cache) et `go test -race` sont les plus lents ;
  `govulncheck` dépend du réseau. Avant chaque commit : `fmt`, `vet`, `test` ; le reste en CI.
- Sur un premier passage, les avertissements les plus fréquents sont `errcheck` (erreurs de
  `Close`, `Fprintf` ignorées) et `revive` (commentaires de documentation manquants). Les erreurs
  de `fmt.Fprintf` vers `os.Stdout` sont un cas classique de `//nolint:errcheck // écriture
  terminal, rien à faire en cas d'échec` — ou d'exclusion configurée une fois pour toutes.

## Défi — Écrire son propre analyseur

```go
var Analyzer = &analysis.Analyzer{
	Name:     "nowcall",
	Doc:      "signale time.Now() hors de main et des tests : injecter l'horloge",
	Requires: []*analysis.Analyzer{inspect.Analyzer},
	Run:      run,
}

func run(pass *analysis.Pass) (any, error) {
	if pass.Pkg.Name() == "main" {
		return nil, nil
	}
	ins := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)
	ins.Preorder([]ast.Node{(*ast.CallExpr)(nil)}, func(n ast.Node) {
		call := n.(*ast.CallExpr)
		fn := typeutil.StaticCallee(pass.TypesInfo, call)
		if fn == nil || fn.Pkg() == nil || fn.Pkg().Path() != "time" || fn.Name() != "Now" {
			return
		}
		if strings.HasSuffix(pass.Fset.File(call.Pos()).Name(), "_test.go") {
			return
		}
		pass.Reportf(call.Pos(), "time.Now() caché : recevoir l'heure en paramètre")
	})
	return nil, nil
}
```

`typeutil.StaticCallee` résout la fonction **réellement** appelée grâce aux informations de type :
un `time.Now` importé sous un alias (`t "time"`) est trouvé, une fonction locale nommée `Now` ne
l'est pas — ce qu'une recherche textuelle (`grep`) ne saurait pas faire.

Une règle mérite un analyseur quand elle est **mécanique** (vérifiable sans jugement), qu'elle a
déjà été enfreinte plusieurs fois, et que son non-respect coûte cher à découvrir plus tard. Une
règle qui demande du jugement (« des noms clairs ») reste dans le guide et la revue.

## Quiz

1. `go vet` ne signale que des erreurs quasi certaines ; `golangci-lint` agrège de nombreux
   linters, dont certains de style, avec plus de faux positifs et une configuration à choisir.
2. Il applique des modernisateurs dont les corrections sont sûres ; `go fix -diff ./...`.
3. Elle demande à `go fix` de remplacer les appels à la fonction annotée par son corps : la
   migration d'une API dépréciée devient automatique.
4. L'usage d'un identifiant déprécié.
5. Parce qu'il est compilé avec une ancienne version de Go et ne sait pas lire les données de
   type (ou la syntaxe) de la nouvelle.
6. Trop d'avertissements, souvent de style ou contradictoires : l'équipe finit par tous les
   ignorer, y compris les vrais.
7. Il ne signale une vulnérabilité que si le code vulnérable est effectivement atteint par le
   graphe d'appels.
8. Toujours ciblé (un linter, une ligne) et justifié en commentaire.
9. Avant commit : `gofmt`, `go vet`, `go test -race`. En CI : tout, y compris `golangci-lint`,
   `go fix -diff` et `govulncheck`.
10. Non : les analyseurs ne voient qu'une partie des problèmes ; les tests et la revue restent
    indispensables.
