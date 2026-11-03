# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Rendre le script plus utile

```bash
#!/usr/bin/env bash
# ci.sh [-k] : les contrôles de la CI. -k : tout exécuter, puis résumer les échecs.
set -uo pipefail
keep=false
[ "${1:-}" = -k ] && keep=true
failed=()

check() { # check "nom" commande…
	local name=$1
	shift
	printf '== %-28s' "$name"
	if out=$("$@" 2>&1); then
		echo ok
	else
		echo ÉCHEC
		printf '%s\n' "$out" | head -20
		failed+=("$name")
		$keep || exit 1
	fi
}

tidy() { go mod tidy && git diff --quiet -- go.mod go.sum; }
fmtcheck() { local u; u=$(gofmt -l .); [ -z "$u" ] || { echo "à formater : $u"; return 1; }; }
vet() { go vet ./... && go vet -tags integration ./...; }
tests() { go test -race -shuffle=on -count=1 -coverprofile=cover.out ./... > test.log 2>&1 || { cat test.log; return 1; }; }
coverage() { # E3 : un seuil par paquet, lu dans la sortie de go test (pondérée par instructions)
	local pct
	pct=$(awk '$2 == "example.com/svc/internal/stats" { for (i = 1; i <= NF; i++) if ($i ~ /%$/) print $i }' test.log)
	echo "internal/stats : $pct"
	awk -v p="${pct%\%}" 'BEGIN { exit !(p + 0 >= 90) }'
}

check "go mod tidy (inchangé)" tidy
check "gofmt" fmtcheck
check "go vet (avec et sans tags)" vet
check "staticcheck" go run honnef.co/go/tools/cmd/staticcheck@v0.8.2 ./...
check "go test -race" tests
check "couverture internal/stats" coverage
check "govulncheck" go run golang.org/x/vuln/cmd/govulncheck@v1.6.0 ./...
check "go build" env CGO_ENABLED=0 go build -buildvcs=true -trimpath -ldflags "-s -w" -o bin/svc ./cmd/svc

if [ ${#failed[@]} -gt 0 ]; then
	echo "échecs : ${failed[*]}"
	exit 1
fi
```

Avec un fichier mal formaté et une fonction non testée ajoutés au paquet `stats` :

```
$ ./ci.sh -k
== go mod tidy (inchangé)     ok
== gofmt                       ÉCHEC
à formater : internal/stats/mal.go
== go vet (avec et sans tags)  ok
== staticcheck                 ok
== go test -race               ok
== couverture internal/stats   ÉCHEC
internal/stats : 87.5%
== govulncheck                 ok
== go build                    ok
échecs : gofmt couverture internal/stats
$ echo $?
1
$ ./ci.sh
== go mod tidy (inchangé)     ok
== gofmt                       ÉCHEC
à formater : internal/stats/mal.go
```

- **`check` encapsule chaque contrôle.** Il capture la sortie, ne l'affiche qu'en cas d'échec, et soit
  s'arrête, soit note l'échec et continue. `set -e` a été retiré, puisque c'est `check` qui décide. Les
  contrôles faits de plusieurs commandes deviennent des fonctions (`tidy`, `vet`).
- **En local**, `-k` donne en une fois la liste de tout ce qu'il faut corriger.
- **En CI**, l'arrêt au premier échec reste le bon comportement par défaut. Il donne le résultat le plus
  vite, et il épargne les étapes qui n'ont pas de sens après un échec : tester du code qui ne passe pas
  `go vet`, publier un binaire dont les tests échouent. Si l'on veut tout voir en CI, on sépare les contrôles
  en **jobs** parallèles plutôt qu'en `-k`.

## E2 — Le test qui dépend de l'ordre

```go
var registry = map[string]int{}

func TestRegister(t *testing.T) {
	registry["a"] = 1
}

func TestLookup(t *testing.T) { // dépend de TestRegister
	if registry["a"] != 1 {
		t.Fatal("a absent")
	}
}
```

```
$ go test -count=1 .              # trois fois : ordre du fichier
ok  	e2o	0.004s
ok  	e2o	0.003s
ok  	e2o	0.003s
$ go test -shuffle=on -count=1 -v .   # six fois
-test.shuffle 1793727041538042540 FAIL
-test.shuffle 1793727041732120669 FAIL
-test.shuffle 1793727041923485819 FAIL
-test.shuffle 1793727042117088803 ok
-test.shuffle 1793727042308761525 ok
-test.shuffle 1793727042499911224 FAIL
```

Sans mélange, les tests passent toujours : ils sont exécutés dans l'ordre du fichier. Avec le mélange, ils
échouent 4 fois sur 6. La graine affichée permet de **rejouer** exactement l'ordre qui a échoué :

```
$ go test -shuffle=1793727041538042540 -count=1 -v .
=== RUN   TestLookup
--- FAIL: TestLookup (0.00s)
=== RUN   TestRegister
FAIL
```

La correction : chaque test prépare **son** état, sans variable de paquet partagée. Après correction, cinq
exécutions mélangées passent toutes.

```go
func newRegistry() map[string]int { return map[string]int{} }

func TestLookup(t *testing.T) {
	r := newRegistry()
	r["a"] = 1 // l'état dont le test a besoin, préparé par le test lui-même
	if r["a"] != 1 {
		t.Fatal("a absent")
	}
}
```

## E3 — La couverture comme garde-fou

La fonction `coverage` du script de E1 lit la couverture **du paquet** dans la sortie de `go test`. C'est un
pourcentage pondéré par les instructions : 87,5 % après l'ajout d'une fonction non testée. Le contrôle échoue
sous 90 %.

Une première version faisait la moyenne des pourcentages par fonction de `go tool cover -func`, et donnait
60 % pour le même code. Une fonction d'une ligne y pesait autant qu'une fonction de cinquante. Il faut savoir
ce que l'on mesure avant d'en faire une règle.

Un seuil unique de 80 % pour tout le dépôt n'est **pas** une bonne idée.

- **Il traite pareil** le paquet métier, où chaque branche compte, et `cmd/svc/main.go` (0 % ici), dont le
  câblage se vérifie par des tests de bout en bout.
- **Il pousse à écrire des tests** qui exécutent du code sans rien vérifier.

Il vaut mieux :

- des seuils **par paquet** pour le code important ;
- ou une règle « la couverture ne **baisse** pas », qui compare avec celle de `main` ;
- et surtout la revue des tests eux-mêmes.

La couverture dit ce qui **n'est pas** testé, jamais ce qui est **bien** testé (N5 L5).

## E4 — L'injection

```yaml
name: pr-title

on: pull_request

permissions:
  contents: read

jobs:
  title:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - name: Afficher et vérifier le titre
        env:
          TITLE: ${{ github.event.pull_request.title }} # une donnée, jamais du code
        run: |
          printf 'Titre : %s\n' "$TITLE"
          if ! printf '%s' "$TITLE" | grep -Eq '^[a-z]+(\([^)]+\))?!?: .+'; then
            echo "::error::le titre ne suit pas la convention type(portée): résumé (leçon 1)"
            exit 1
          fi
```

```
$ actionlint && echo "actionlint : aucun problème"
actionlint : aucun problème
```

La valeur passe par `env:` : GitHub la place dans une variable d'environnement, et le shell la lit comme une
**donnée** (`"$TITLE"`). `printf '%s'` évite en outre qu'un titre commençant par `-` soit lu comme une option.

La version dangereuse, `run: echo "Titre ${{ github.event.pull_request.title }}"`, fait remplacer
l'expression par GitHub **dans le texte du script**, avant que le shell ne le lise. Avec le titre
`x"; echo PWNED; echo "`, voici le script obtenu et son exécution, simulés localement :

```
$ bash -c 'echo "Titre x"; echo PWNED; echo ""'
Titre x
PWNED

```

Le guillemet du titre ferme celui du script, et la suite devient une commande. À la place de `echo PWNED`,
un attaquant écrirait une commande qui envoie les secrets du job, ou le jeton s'il a des droits d'écriture.

`actionlint` le signale (cours), et la variante sûre affiche simplement :

```
Titre : x"; echo PWNED; echo "
```

## E5 — Épingler et mettre à jour

```
$ git ls-remote https://github.com/actions/checkout.git refs/tags/v7.0.1 'refs/tags/v7.0.1^{}'
3d3c42e5aac5ba805825da76410c181273ba90b1	refs/tags/v7.0.1
$ git ls-remote https://github.com/softprops/action-gh-release.git refs/tags/v3.0.3 'refs/tags/v3.0.3^{}'
e598afbe1493e6b1bafb1f389cabb956eab91231	refs/tags/v3.0.3
efb35369e0ad2afab669f228072c1b0d510eae64	refs/tags/v3.0.3^{}
```

- **Une étiquette légère** (`checkout`) ne donne qu'une ligne : c'est directement l'empreinte du commit.
- **Une étiquette annotée** (`action-gh-release`) donne deux lignes. La première est l'empreinte de l'**objet
  étiquette** ; la seconde, suffixée `^{}`, celle du **commit** qu'elle désigne. C'est la seconde qu'on
  épingle : `uses:` attend une empreinte de commit.

C'est pourquoi le cours utilise `efb35369…` pour `action-gh-release`, et non `e598afbe…`.

Pour être prévenu des nouvelles versions, on ajoute `.github/dependabot.yml`. Dependabot ouvre alors des *pull
requests* qui mettent à jour l'empreinte **et** le commentaire de version. La CI les vérifie comme n'importe
quel changement.

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
  - package-ecosystem: gomod
    directory: /
    schedule:
      interval: weekly
```

L'écosystème `gomod` fait de même pour les dépendances Go (`go.mod`), ce qui complète `govulncheck`
(leçon 8).

---

## Exercice intermédiaire — La CI du service de prêts

> Cette partie se fait dans **ton** dépôt GitHub de test : elle n'a pas été exécutée sur GitHub pour ce
> corrigé. Les fichiers ont été validés par `actionlint`.

1. `ci.sh` : celui de E1. On y ajoute `go vet -tags <étiquette>` pour chaque étiquette utilisée dans le
   service (`grep -rh '^//go:build' --include=*.go . | sort -u` les liste).
2. Le job `integration` du cours, avec la commande de la suite de contrat :

   ```yaml
         - run: go test -race -tags integration -count=1 ./internal/...
           env:
             DATABASE_URL: postgres://ci:ci@localhost:5432/ci?sslmode=disable
   ```

   Côté Go, la suite de contrat (N8 L7) ne doit **pas** se sauter si la base manque quand l'étiquette est
   présente. On applique le même principe que `TestDatabaseReachable` : `t.Fatal`, pas `t.Skip`.
3. La course volontaire : un compteur partagé incrémenté sans verrou dans un handler, et un test qui lance
   des requêtes concurrentes. Le job `checks` échoue sur `WARNING: DATA RACE`, et la *pull request* affiche
   une croix rouge. Elle ne peut pas être fusionnée si le contrôle est requis.
4. *Settings → Branches* (ou *Rules → Rulesets*) :
   - exiger une *pull request* avant la fusion ;
   - exiger les contrôles `checks` et `integration`, et que la branche soit à jour ;
   - interdire les poussées forcées et la suppression de la branche.

   Les noms à cocher sont ceux des **jobs**. Un job renommé n'est plus requis, et la protection
   disparaît en silence : c'est un point à vérifier à chaque changement de workflow.
5. La mesure de durée se lit dans l'onglet *Actions*, job par job. On s'attend à ce que la différence porte
   surtout sur le téléchargement des modules et la recompilation de la bibliothèque standard sous `-race` :
   c'est ce que montre la leçon 3, où le cache de compilation divisait le temps par 10. On n'a pas pu le
   mesurer sur GitHub pour ce corrigé : c'est ta mesure qui fait foi.

---

## Défi — La publication de bout en bout

> Comme l'exercice intermédiaire, ce défi se fait dans ton dépôt de test, et n'a pas été exécuté pour ce
> corrigé. Voici ce qu'il faut observer, et pourquoi.

**1.**

- **`v0.1.0-rc.1`** donne une publication marquée *Pre-release* (`contains(github.ref_name, '-')`).
- **`v0.1.0`** donne une publication normale. Le générateur de notes (`generate_release_notes`) liste les
  *pull requests* fusionnées depuis l'étiquette précédente.
- **L'image** : `docker buildx imagetools inspect ghcr.io/<toi>/<dépôt>:0.1.0` doit lister deux manifestes,
  `linux/amd64` et `linux/arm64`.

**2. La reconstruction.** Le binaire est reproductible (leçon 2, défi) si tout ce qui entre dans le build est
identique.

- **La version de Go** est fixée par la ligne `toolchain`.
- **Les options** sont fixées par `build.sh`.
- **Les informations Git** dépendent de l'état du dépôt : c'est le point délicat. En CI, `checkout` récupère
  le commit de l'étiquette, dans un dépôt propre. Localement, il faut se placer **sur l'étiquette**
  (`git checkout v0.1.0`), dans un arbre **sans aucune modification ni fichier non suivi** (`+dirty`,
  leçon 1). Un `dist/` non ignoré suffit à tout changer.
- **La date**, `git log -1 --format=%cs`, est celle du commit, identique partout.

Si les sommes diffèrent, `go version -m` sur les deux binaires montre la ligne qui diffère, le plus souvent
`vcs.modified` ou `vcs.revision`.

**3. La provenance.** `provenance: mode=max` dans `docker/build-push-action` fait produire par BuildKit une
**attestation** jointe à l'image. Elle décrit comment l'image a été construite : le dépôt et le commit
source, le workflow, les arguments de build, les images de base et leurs empreintes. Elle se lit avec `docker buildx imagetools inspect`, dont l'option `--format` permet d'extraire la provenance. La syntaxe exacte est dans la documentation de la commande, et n'a pas été essayée ici.

C'est la base pour vérifier qu'une image vient bien de **ce** dépôt et de **ce** commit. La leçon 8 y
ajoute la signature.

**4. Les droits.**

| Job | Droits | Pourquoi |
|---|---|---|
| `ci.yml`, tous les jobs | `contents: read` | lire le dépôt, rien d'autre |
| `release.yml` → `binaries` | `contents: write` | créer la publication GitHub et y joindre les fichiers |
| `release.yml` → `image` | `contents: read`, `packages: write` | pousser l'image dans `ghcr.io` |

Aucun job n'a besoin d'`actions: write`, de `pull-requests: write` ou d'`id-token: write`. Le dernier
deviendrait nécessaire pour signer sans clé (leçon 8). Le jeton n'est jamais exposé à du code venu d'une
*pull request* externe : un workflow `pull_request` lancé depuis une *fork* n'a pas accès aux secrets, et le workflow de publication ne se
déclenche que sur une étiquette, que seuls les mainteneurs peuvent pousser.

---

## Quiz

1. Du plus rapide au plus lent : `go mod tidy` sans différence, `gofmt`, `go vet` (avec chaque étiquette),
   `staticcheck`, `go test -race -shuffle=on`, `govulncheck`, puis le build.
2. Une course de données ne fait pas forcément échouer un test : 8000 = 8000 trois fois de suite, avec une
   course. Seul le détecteur la voit.
3. Il exécute les tests (et les benchmarks) dans un ordre aléatoire, et affiche la graine pour rejouer cet
   ordre. Les dépendances cachées entre tests sont démasquées.
4. Pour exécuter exactement la même chose en local et en CI, déboguer la CI sur son poste, et ne pas
   enfermer la logique dans un format propre à un fournisseur.
5. `actions/setup-go` avec `go-version-file: go.mod` : la ligne `toolchain` si elle existe, sinon la ligne
   `go`.
6. Avec un `services:` (un conteneur PostgreSQL et sa sonde de santé), des tests derrière `//go:build
   integration`, et l'adresse passée par variable d'environnement. Ces tests échouent, et ne se sautent pas,
   si la base manque.
7. Une étiquette peut être déplacée par l'auteur, ou par un attaquant qui a pris son compte. Une empreinte de
   commit désigne un code immuable.
8. L'expression est remplacée **dans le texte du script** avant son exécution : un titre peut fermer les
   guillemets et injecter des commandes. On passe la valeur par `env:` et on l'utilise en `"$VARIABLE"`.
9. Pour que le jeton du workflow n'ait que les droits nécessaires : lecture seule par défaut, écriture
   seulement pour le job qui publie.
10. Relancer tous les contrôles sur le commit étiqueté. On ne publie que ce qui les passe.
