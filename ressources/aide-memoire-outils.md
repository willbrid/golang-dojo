# Aide-mémoire — la chaîne d'outils Go

Toutes ces commandes s'expliquent aussi avec `go help <commande>`.

## Le quotidien

| Commande | Effet |
|---|---|
| `go run .` | Compile dans un répertoire temporaire **et** exécute. Ne produit pas de binaire. |
| `go build` | Compile et écrit le binaire dans le répertoire courant. |
| `go build -o bin/app .` | Idem, à l'emplacement choisi. |
| `go test ./...` | Lance tous les tests du module. |
| `go test -run TestFoo -v ./...` | Un seul test, en mode verbeux. |
| `go test -race ./...` | Avec le détecteur de data races. **Indispensable dès le niveau 4.** |
| `go test -cover ./...` | Avec la couverture. |
| `go fmt ./...` | Reformate selon le style officiel. Aucune option. |
| `go vet ./...` | Analyse statique : bugs probables que le compilateur laisse passer. |
| `go doc fmt.Sprintf` | Documentation dans le terminal. |
| `go doc -all strings` | Toute l'API d'un paquet. |

`./...` = « ce répertoire et tous ses sous-répertoires ».

## Modules

| Commande | Effet |
|---|---|
| `go mod init <chemin>` | Crée `go.mod`. Le chemin est l'URL du dépôt, sans `https://`. |
| `go mod tidy` | Ajoute les dépendances manquantes, retire les inutiles. |
| `go get exemple.com/pkg@v1.2.3` | Ajoute ou change une dépendance. |
| `go get -u ./...` | Met à jour les dépendances. |
| `go list -m all` | Liste le graphe de modules effectif. |
| `go mod why exemple.com/pkg` | Explique pourquoi une dépendance est là. |

## Diagnostic

| Commande | Effet |
|---|---|
| `go env` | Toutes les variables d'environnement Go. |
| `go env GOROOT GOPATH GOMODCACHE` | Installation / espace de travail / cache des modules. |
| `go version -m ./bin/app` | Version de Go et dépendances d'un binaire compilé. |
| `go tool pprof` | Analyse de profils (niveau 8). |
| `go tool trace` | Trace d'exécution. Depuis 1.27, `-http` n'écoute que sur localhost. |
| `govulncheck ./...` | Vulnérabilités connues (à installer). |

## Spécifique à Go 1.27

| Commande | Effet |
|---|---|
| `go fix ./...` | Modernise le code. Nouveaux passages : `atomictypes`, `embedlit`, `slicesbackward`, `unsafefuncs`. |
| `go doc pkg@version` | Documentation d'une version précise d'un module. |
| `go doc -ex` | Affiche les exemples exécutables. |
| `go test` | Lance désormais la vérification `stdversion` par défaut. |

## Cross-compilation

```bash
GOOS=linux   GOARCH=amd64 go build -o bin/app-linux   .
GOOS=darwin  GOARCH=arm64 go build -o bin/app-mac     .
GOOS=windows GOARCH=amd64 go build -o bin/app.exe     .
go tool dist list        # toutes les cibles possibles
```

## Liens officiels

- <https://go.dev/doc/> — documentation
- <https://go.dev/ref/spec> — spécification du langage
- <https://pkg.go.dev/> — documentation des paquets
- <https://go.dev/blog/> — le blog Go
- <https://go.dev/doc/effective_go> — Effective Go
- <https://go.dev/wiki/CodeReviewComments> — critères de revue de code Go
- <https://go.dev/doc/go1.27> — notes de version 1.27
