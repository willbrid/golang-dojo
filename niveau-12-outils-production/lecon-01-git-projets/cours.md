# Leçon 1 — Git et projets Go

> Pour Go, un dépôt Git n'est pas seulement un endroit où ranger le code. Les **étiquettes** sont les
> versions des modules (N2 L9), et `go build` lit l'état du dépôt pour l'inscrire dans le binaire. Une
> étiquette publiée devient immuable pour le monde entier. Cette leçon couvre ce qu'un développeur Go doit
> savoir de Git : organiser le dépôt, écrire des commits utiles, publier des versions, retrouver la version
> d'un binaire en production, et trouver le commit qui a introduit une régression.

## Objectifs

- Organiser un dépôt Go, et savoir ce qu'on y valide et ce qu'on ignore.
- Travailler en branches courtes sur une branche principale toujours saine.
- Écrire des messages de commit qui servent à la revue, à l'historique et aux notes de version.
- Étiqueter des versions, préversions et sous-modules, et savoir pourquoi une étiquette ne se déplace
  jamais.
- Lire la version, le commit et l'état du dépôt inscrits dans un binaire.
- Trouver une régression avec `git bisect run go test`.

## Explication

### L'arborescence d'un dépôt Go

```
.
├── go.mod, go.sum        à la racine : un dépôt = un module, dans le cas courant
├── cmd/
│   └── app/main.go       un répertoire par exécutable
├── internal/             le code non importable de l'extérieur (N10 L7)
│   └── price/
├── testdata/             données de test, ignorées par l'outil go
├── .gitignore
└── README.md
```

Ni `src/`, ni `pkg/` obligatoire : l'outil `go` n'impose que `go.mod`. Le chemin d'un paquet est celui de son
répertoire sous le module.

**Ce qu'on valide :** `go.mod` **et** `go.sum`. Le second garantit que tout le monde compile les mêmes octets
(N2 L9). On valide aussi le code généré, si `go generate` n'est pas lancé par le build (N9 L4), et les
`testdata`.

**Ce qu'on ignore** (`.gitignore`) :

- les **binaires** (`/bin/`) ;
- les binaires de test (`*.test`) ;
- les profils et la couverture (`*.out`, leçon 4 du niveau 11) ;
- les fichiers de secrets locaux (`.env`).

Un binaire non ignoré a aussi un effet sur la version : voir plus bas.

Le répertoire `vendor/` (une copie des dépendances, `go mod vendor`) n'est utile que si l'on doit construire
sans accès au réseau ni au proxy. Par défaut, on ne le valide pas.

### Les branches

La pratique la plus répandue pour un service est le développement **sur le tronc** :

- `main` est **toujours** déployable : les tests passent, le build réussit ;
- chaque changement se fait sur une branche **courte** (quelques heures à quelques jours), qui part de
  `main` et y revient par une demande de fusion (*pull request*) relue et validée par la CI (leçon 4) ;
- `main` est **protégée** : pas de poussée directe, CI verte obligatoire avant la fusion.

Les branches longues accumulent des conflits et retardent les problèmes d'intégration. Une fonctionnalité
inachevée se fusionne désactivée, derrière un drapeau de configuration (leçon 5), plutôt que de vivre trois
semaines sur une branche.

### Les messages de commit

Un message de commit est lu en revue, dans `git log`, dans `git blame` quand on cherche pourquoi une ligne
existe, et dans les notes de version. Sa forme :

```
type(portée): résumé à l'impératif, moins de 72 caractères

Le corps explique POURQUOI, pas comment (le diff montre le comment) :
le problème, la décision, ce qui a été écarté.

Refs: #142
```

- **Les types courants** sont `feat` (fonctionnalité), `fix` (correction), `docs`, `refactor`, `test`, `perf`
  et `build`.
- **Une rupture de compatibilité** s'annonce par `!` (`feat!:`) ou par une ligne `BREAKING CHANGE:` dans le
  corps.

C'est la convention dite des *Conventional Commits*. Elle correspond au versionnement sémantique : un `fix`
appelle une version corrective, un `feat` une version mineure, une rupture une version majeure. Des outils en
tirent les notes de version.

Le projet Go lui-même utilise une forme plus simple, `paquet: résumé` (par exemple
`net/http: fix race in Transport`), décrite dans son guide de contribution (go.dev/doc/contribute). L'une
ou l'autre fera l'affaire. L'important est qu'**une équipe en choisisse une et s'y tienne**.

Un commit = **un** changement cohérent, qui compile et passe les tests. C'est ce qui rend `git bisect`
utilisable (plus bas).

### Les étiquettes de version

Une version d'un module Go **est** une étiquette Git de la forme `vMAJEUR.MINEUR.CORRECTIF` (N2 L9) :

| Étiquette | Sens |
|---|---|
| `v1.4.0` | version publiée |
| `v1.5.0-rc.1` | préversion : `go get module@latest` ne la choisit pas |
| `v2.0.0` | version majeure 2 : le module doit s'appeler `…/v2` (N2 L9) |
| `outils/v0.3.0` | version du module situé dans le sous-répertoire `outils/` d'un dépôt qui en contient plusieurs |

On utilise des étiquettes **annotées** (`git tag -a v1.4.0 -m "…"`), qui portent un auteur, une date et un
message. Les étiquettes légères ne sont qu'un nom sur un commit.

**Une étiquette poussée ne se déplace jamais**, et ne se supprime pas pour être recréée.

- Au premier téléchargement, le proxy public des modules (`proxy.golang.org`) **garde une copie** de la
  version.
- La base de sommes de contrôle (`sum.golang.org`) enregistre son empreinte.

Si l'on déplace l'étiquette, ceux qui ont déjà la version gardent l'ancien code. Ceux qui la téléchargent
directement obtiennent une erreur `checksum mismatch` (sommes de contrôle différentes). On ne corrige pas une
version, on en publie une **nouvelle** (`v1.4.1`) et on retire la mauvaise avec `retract` dans `go.mod`
(N2 L9).

### La version dans le binaire

Depuis Go 1.24, `go build` inscrit dans le binaire la **version du module principal**, tirée de Git :

| État du dépôt au moment du build | Version inscrite |
|---|---|
| sur un commit étiqueté `v1.0.0`, rien de modifié | `v1.0.0` |
| même commit, fichiers modifiés ou non suivis | `v1.0.0+dirty` |
| un commit après `v1.0.0` | une **pseudo-version** `v1.0.1-0.AAAAMMJJhhmmss-empreinte` |
| aucune étiquette dans l'historique | `v0.0.0-AAAAMMJJhhmmss-empreinte` |
| `go run`, ou pas d'informations Git | `(devel)` |

Il inscrit aussi le **commit** (`vcs.revision`), sa date (`vcs.time`) et l'état du dépôt (`vcs.modified`). On
les lit avec `runtime/debug.ReadBuildInfo`, ou de l'extérieur avec `go version -m binaire`.

Une pseudo-version se lit comme une version : `v1.0.1-0.…` est une **préversion** de `v1.0.1`, donc
postérieure à `v1.0.0` et antérieure à `v1.0.1`. Sa date est celle du commit (la date du *committer*), pas
celle du build.

Deux pièges :

- **Un fichier non suivi suffit pour obtenir `+dirty`**. C'est le cas typique du binaire `app` construit à la
  racine sans être ignoré par Git. D'où le `.gitignore`.
- **Sans le programme `git`** (dans une image de build, leçon 3), `go build` réussit sans rien dire et la
  version vaut `(devel)`. Avec `-buildvcs=true`, il échoue (`error obtaining VCS status`), ce qui est
  préférable en CI : on veut savoir que la version est absente. À l'inverse, `-buildvcs=false` omet
  volontairement ces informations.

La leçon 2 complète cette version par des valeurs injectées au build (`-ldflags`).

### Trouver une régression : `git bisect`

Un test qui passait à la version `v1.1.0` échoue sur `main`, cent commits plus loin. `git bisect` fait une
**recherche dichotomique** dans l'historique :

```
git bisect start main v1.1.0               # mauvais, bon
git bisect run go test ./internal/price/   # le code de retour décide : 0 = bon, autre = mauvais
git bisect reset
```

`git bisect run` lance la commande sur un commit au milieu de l'intervalle et garde la moitié coupable.
Pour cent commits, il faut sept essais. Le résultat est **le** commit qui a introduit la régression, avec
son message : d'où l'intérêt de commits petits, qui compilent tous. Un commit qui ne compile pas renvoie un
code d'erreur et fausse la recherche. On peut le faire sauter avec le code 125 (`git bisect skip`) dans un
script.

## Exemple

Un script qui crée un dépôt de démonstration dans un répertoire temporaire. Il montre la version inscrite dans
le binaire à chaque étape, puis retrouve une régression. L'auteur et les dates des commits sont fixés : les
empreintes sont les mêmes à chaque exécution, et chez toi aussi.

```bash
#!/usr/bin/env bash
# demo.sh : la version qu'un binaire Go tire de git, puis git bisect avec go test.
set -euo pipefail
dir=$(mktemp -d)
cd "$dir"

# Auteur et dates fixes : les empreintes des commits sont reproductibles.
export GIT_AUTHOR_NAME=Dev GIT_AUTHOR_EMAIL=dev@example.com
export GIT_COMMITTER_NAME=Dev GIT_COMMITTER_EMAIL=dev@example.com
commit() { # commit "date" "message"
	GIT_AUTHOR_DATE="$1" GIT_COMMITTER_DATE="$1" git commit -q -m "$2"
}
build() { go build -o bin/app ./cmd/app && bin/app; }

git init -q -b main
cat > go.mod <<'GOMOD'
module example.com/app

go 1.27
GOMOD
mkdir -p cmd/app internal/price
cat > cmd/app/main.go <<'GO'
// Commande app : affiche la version que le binaire connaît de lui-même.
package main

import (
	"fmt"
	"runtime/debug"
)

func main() {
	info, ok := debug.ReadBuildInfo()
	if !ok {
		fmt.Println("pas d'informations de construction")
		return
	}
	settings := map[string]string{}
	for _, s := range info.Settings {
		settings[s.Key] = s.Value
	}
	fmt.Printf("version %s, commit %.12s, modifié %s\n",
		info.Main.Version, settings["vcs.revision"], settings["vcs.modified"])
}
GO
cat > internal/price/price.go <<'GO'
package price

// TTC ajoute 20 % de TVA à un prix en centimes.
func TTC(cents int64) int64 { return cents * 120 / 100 }
GO
cat > internal/price/price_test.go <<'GO'
package price

import "testing"

func TestTTC(t *testing.T) {
	if got := TTC(1000); got != 1200 {
		t.Fatalf("TTC(1000) = %d, attendu 1200", got)
	}
}
GO
printf '/bin/\n*.test\n*.out\n.env\n' > .gitignore
git add .
commit "2026-03-01T10:00:00Z" "feat: première version"

echo "== 1. commit sans étiquette"
build
echo "== 2. étiquette v1.0.0"
git tag -a v1.0.0 -m "v1.0.0"
build
echo "== 3. un fichier modifié, pas encore validé"
echo "// TODO" >> internal/price/price.go
build
git checkout -q internal/price/price.go
echo "== 4. un commit après v1.0.0"
echo "// Package price calcule les prix." | cat - internal/price/price.go > p && mv p internal/price/price.go
git commit -qam "docs(price): commentaire de paquet" --date "2026-03-02T10:00:00Z"
GIT_COMMITTER_DATE="2026-03-02T10:00:00Z" git commit -q --amend --no-edit --date "2026-03-02T10:00:00Z"
build
echo "== 5. préversion v1.1.0-rc.1"
git tag -a v1.1.0-rc.1 -m "v1.1.0-rc.1"
build
echo "== 6. go run"
go run ./cmd/app

echo "== 7. git bisect : quel commit a cassé TestTTC ?"
for i in 1 2 3 4 5; do
	if [ "$i" = 3 ]; then
		sed -i 's|cents \* 120 / 100|cents * 12 / 10 + 1|' internal/price/price.go # la régression
	fi
	echo "// étape $i" >> cmd/app/main.go
	git commit -qam "refactor: étape $i"
	GIT_COMMITTER_DATE="2026-03-0$((2 + i))T10:00:00Z" git commit -q --amend --no-edit --date "2026-03-0$((2 + i))T10:00:00Z"
done
git bisect start HEAD v1.1.0-rc.1 > /dev/null
git bisect run go test ./internal/price/ > bisect.log 2>&1
grep "is the first bad commit" bisect.log
grep -A5 "is the first bad commit" bisect.log | grep "    refactor"
git bisect reset > /dev/null 2>&1
```

Sortie (identique d'une exécution à l'autre) :

```
== 1. commit sans étiquette
version v0.0.0-20260301100000-296a52ba0b8b, commit 296a52ba0b8b, modifié false
== 2. étiquette v1.0.0
version v1.0.0, commit 296a52ba0b8b, modifié false
== 3. un fichier modifié, pas encore validé
version v1.0.0+dirty, commit 296a52ba0b8b, modifié true
== 4. un commit après v1.0.0
version v1.0.1-0.20260302100000-29a6f57cb826, commit 29a6f57cb826, modifié false
== 5. préversion v1.1.0-rc.1
version v1.1.0-rc.1, commit 29a6f57cb826, modifié false
== 6. go run
version (devel), commit , modifié 
== 7. git bisect : quel commit a cassé TestTTC ?
ca51e7822cc1e51fc1705f6179552262c299b824 is the first bad commit
    refactor: étape 3
```

## Explication du code

| Élément | Rôle |
|---|---|
| `GIT_AUTHOR_DATE`, `GIT_COMMITTER_DATE` | Fixent les dates du commit. L'empreinte d'un commit dépend de son contenu, de son auteur, de ses dates et de son parent : tout étant fixé, elle est reproductible. |
| `printf … > .gitignore` | `/bin/` ignore le binaire construit par `build()`. Sans cette ligne, l'étape 1 donnerait déjà `+dirty`. |
| `debug.ReadBuildInfo()` | Renvoie la version du module principal (`info.Main.Version`) et les réglages du build (`info.Settings`), dont les clés `vcs.*`. |
| Étape 3 | Un fichier modifié et non validé : la version devient `v1.0.0+dirty`, et `vcs.modified` vaut `true`. On n'envoie **jamais** un tel binaire en production : il ne correspond à aucun commit. |
| Étape 4 | Un commit après l'étiquette : pseudo-version `v1.0.1-0.20260302100000-29a6f57cb826`, avec la date du commit et les 12 premiers caractères de son empreinte. |
| Étape 6 | `go run` n'inscrit pas les informations Git : `(devel)`, et un commit vide. |
| `git commit --amend` avec `GIT_COMMITTER_DATE` | Fixe aussi la date du *committer*, que `git commit --date` ne change pas (c'est elle que la pseudo-version utilise). |
| `git bisect run go test ./internal/price/` | `go test` renvoie 0 si les tests passent, 1 sinon : exactement le contrat attendu par `bisect run`. Le commit coupable, « étape 3 », est trouvé en trois essais pour cinq commits. |

## Erreurs fréquentes

1. **Ne pas valider `go.sum`**, ou valider les binaires.
2. **Déplacer ou recréer une étiquette déjà poussée** pour « corriger » une version.
3. **Oublier `/v2`** en publiant `v2.0.0` (N2 L9).
4. **Des messages de commit vides de sens** (« fix », « wip », « modifs ») : la revue, `blame` et `bisect`
   perdent toute valeur.
5. **Des commits qui ne compilent pas** au milieu d'une série : `bisect` ne peut plus conclure.
6. **Déployer un binaire `+dirty`**, ou `(devel)` : impossible de savoir quel code tourne.
7. **Des branches de plusieurs semaines**, et la fusion douloureuse qui suit.

## Bonnes pratiques Go

- `go.mod` à la racine, `cmd/<nom>/` par exécutable, `internal/` pour le reste.
- Un `.gitignore` qui couvre binaires, `*.test`, `*.out`, `.env`.
- `main` protégée et toujours saine ; branches courtes ; CI obligatoire (leçon 4).
- Une convention de messages, la même pour toute l'équipe ; un commit = un changement qui compile.
- Étiquettes annotées `vX.Y.Z`, jamais déplacées ; préversions `-rc.N` pour tester une version avant de la
  publier.
- Le service affiche sa version et son commit au démarrage, et les expose (leçons 6 et 9).
- `go build -buildvcs=true` en CI, pour échouer si la version ne peut pas être inscrite.

## Ce que je dois retenir

- Une étiquette Git `vX.Y.Z` **est** la version du module ; publiée, elle est immuable.
- `go build` inscrit la version, le commit et l'état du dépôt dans le binaire : `+dirty` dès qu'un fichier
  n'est pas validé, `(devel)` avec `go run` ou sans Git.
- Les pseudo-versions `vX.Y.(Z+1)-0.date-empreinte` désignent un commit postérieur à `vX.Y.Z`.
- Des commits petits, cohérents et bien décrits servent la revue, l'historique et `git bisect`.
- `git bisect run go test ./…` trouve le commit d'une régression par dichotomie.

---

➡️ [Exercices](exercices.md)
