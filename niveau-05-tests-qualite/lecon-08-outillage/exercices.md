# Leçon 8 — Exercices

Code dans `mes-solutions/niveau-05/lecon-08/`.

## Exercices faciles

**E1 — L'état de ta machine.**
Pour chaque outil installé dans `$(go env GOPATH)/bin`, afficher la version de Go avec laquelle il
a été construit (`go version -m <binaire> | head -1`). Lesquels sont plus anciens que ton `go
version` ? Lancer `staticcheck ./...` et `golangci-lint run` sur un petit module en `go 1.27` :
fonctionnent-ils ? Réinstaller ceux qui échouent, et relancer.

**E2 — `go vet`.**
Écrire un fichier qui déclenche au moins **cinq** analyseurs différents de `go vet` (indices :
`printf`, `copylocks`, `unreachable`, `structtag`, `unusedresult`, `loopclosure`…). Pour chacun,
recopier le message. Lequel aurais-tu pu ne pas voir en relecture ?

**E3 — Moderniser.**
Lancer `go fix -diff ./...` sur ton code du projet 1 (`wordstat`) et du projet 2. Combien de
modernisations ? Les appliquer dans un commit séparé, puis vérifier que tous les tests passent
encore. Quel modernisateur a fait le plus de changements ?

**E4 — `//go:fix inline`.**
Dans un petit module, déclarer une fonction dépréciée et son remplaçant :

```go
// Deprecated: utiliser NewClient.
//
//go:fix inline
func Connect(addr string) *Client { return NewClient(addr, DefaultTimeout) }
```

Dans un autre paquet qui appelle `Connect`, lancer `go fix -diff ./...`. Que propose-t-il ?
En quoi ce mécanisme change-t-il la vie des **utilisateurs** d'une bibliothèque ?

**E5 — Le tableau de l'exemple.**
Corriger **tous** les problèmes du programme d'exemple du cours, y compris ceux qu'aucun outil ne
trouve. Écrire un test qui aurait trouvé la panique sur `cfg` nil.

---

## Exercice intermédiaire — La chaîne qualité du projet 2

Mettre en place, pour le projet `todo` :

1. Un `Makefile` (ou un script `check.sh`) avec une cible `check` qui lance, dans l'ordre et en
   s'arrêtant au premier échec : `gofmt -l` (échec si non vide), `go vet ./...`, `go fix -diff`
   (échec si non vide), `golangci-lint run`, `go test -race -shuffle=on ./...`, `govulncheck ./...`.
2. Un `.golangci.yml` partant de la configuration du cours. Lancer : combien d'avertissements ?
   Les traiter tous : corriger, ou `//nolint:… // raison` si l'avertissement est injustifié.
3. Activer **un** linter supplémentaire de ton choix (`gocritic`, `nilerr`, `exhaustive`…) : qu'a-t-il
   trouvé ? Le garder ou non ? Justifier.
4. Mesurer le temps total de `make check`. Quelle étape est la plus lente ? Que lancer avant
   chaque commit, et que réserver à la CI ?

---

## Défi — Écrire son propre analyseur

Le paquet `golang.org/x/tools/go/analysis` permet d'écrire des analyseurs qui s'exécutent avec
`go vet -vettool`.

1. Écrire un analyseur `nowcall` qui signale tout appel à `time.Now()` en dehors des paquets
   `main` et `_test` — la règle de la leçon 3 (« pas d'horloge cachée »).
2. Le tester avec `analysistest` (des fichiers `testdata/src/…` annotés `// want "…"`).
3. Le lancer sur le projet 2 avec `go vet -vettool=$(which nowcall) ./...`. Trouve-t-il quelque
   chose ?
4. En quelques lignes : quand une règle d'équipe mérite-t-elle un analyseur plutôt qu'une ligne
   dans un guide de style ?

---

## Quiz

1. Quelle différence de philosophie entre `go vet` et `golangci-lint` ?
2. Que fait `go fix` depuis Go 1.26 ? Comment voir ses changements sans les appliquer ?
3. Que fait la directive `//go:fix inline` ?
4. Que signifie l'identifiant `SA1019` de `staticcheck` ?
5. Pourquoi un linter peut-il cesser de fonctionner après une mise à jour de Go ?
6. Pourquoi ne pas activer tous les linters de `golangci-lint` ?
7. Qu'est-ce qui distingue `govulncheck` d'un simple scanner de dépendances ?
8. Quelle est la règle pour `//nolint` ?
9. Quels outils lancer avant chaque commit, et lesquels en CI ?
10. Un code qui passe tous les analyseurs est-il correct ?
