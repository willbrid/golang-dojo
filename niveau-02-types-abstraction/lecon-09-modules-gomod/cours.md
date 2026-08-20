# Leçon 9 — Modules et `go.mod`

## Objectifs

1. Comprendre ce qu'est un module, et sa relation avec les packages et le dépôt.
2. Lire et écrire un `go.mod` et un `go.sum` en sachant ce que chaque ligne signifie.
3. Ajouter, mettre à jour et retirer une dépendance en maîtrisant le versionnement sémantique.
4. Comprendre la sélection de version minimale (MVS), spécificité de Go.

## Explication

### Module, package, dépôt

```
DÉPÔT git  ─ contient généralement ─→  1 MODULE  ─ contient ─→  N PACKAGES
                                       (go.mod)                 (répertoires)
```

Un **module** est l'unité de **versionnement et de distribution**. Il est défini par un
`go.mod` à la racine, et son nom est le **chemin d'import racine** de tous ses packages.

```
module github.com/willbrid/biblio

go 1.27
```

Un dépôt peut contenir plusieurs modules (un `go.mod` par sous-répertoire), mais c'est
inhabituel et compliqué à versionner : **un dépôt, un module** est la règle par défaut.

### Anatomie de `go.mod`

```
module github.com/willbrid/biblio      // chemin d'import racine

go 1.27                                 // version du LANGAGE demandée

require (
	github.com/google/uuid v1.6.0       // dépendance directe
	golang.org/x/text v0.14.0           // indirect
)

require (
	github.com/kr/pretty v0.3.1 // indirect
)

replace github.com/x/y => ../y          // redirection LOCALE, à ne jamais committer
exclude github.com/z/w v1.2.3           // interdit une version précise
retract v1.0.1                          // je retire MA version publiée : elle est cassée
```

Points à connaître :

- **`go 1.27` n'est pas une version minimale d'installation.** C'est la version du **langage**
  demandée : elle active les fonctionnalités correspondantes et fixe la sémantique — la
  variable de boucle de 1.22 en est l'exemple le plus concret (niveau 1, leçon 5). Depuis
  Go 1.21, si le toolchain local est plus ancien que cette ligne, Go **télécharge
  automatiquement** le bon toolchain.
- **`// indirect`** signale une dépendance qui n'est pas importée directement par le code du
  module : elle vient d'une dépendance de dépendance. `go mod tidy` gère ces marqueurs.
- **`replace`** est très utile en développement local (travailler sur deux modules en
  parallèle) et **catastrophique** si on l'oublie avant de publier : les utilisateurs du
  module hériteraient d'un chemin qui n'existe pas chez eux. Pour un usage local, préférer
  `go.work` (voir plus bas).
- **`retract`** permet à un auteur de signaler qu'une de ses versions publiées ne doit plus
  être utilisée. Comme les versions Go sont immuables et mises en cache pour toujours, c'est
  le seul moyen de « rappeler » une version.

### `go.sum` : l'intégrité, pas les versions

`go.sum` contient les **empreintes cryptographiques** de chaque module utilisé. Ce n'est pas
un fichier de verrouillage de versions (c'est le rôle de `go.mod`) : c'est une garantie que le
contenu téléchargé est exactement celui qui a été vu la première fois.

**`go.sum` se committe toujours.** Il est vérifié à chaque build. Un module dont l'empreinte
ne correspond plus provoque une erreur de sécurité explicite — c'est ce qui protège contre la
substitution de code en amont.

Deux services publics interviennent : le **proxy** (`proxy.golang.org`), qui met en cache les
modules pour toujours, et la **base de sommes de contrôle** (`sum.golang.org`), qui les
authentifie. Ils se configurent par `GOPROXY`, `GOSUMDB` et `GOPRIVATE` — cette dernière étant
indispensable pour les dépôts d'entreprise privés.

### Le versionnement sémantique, et la règle du `/v2`

Go impose le **semver** : `vMAJEUR.MINEUR.CORRECTIF`.

| Changement | Incrément | Signification |
|---|---|---|
| correction de bug | `v1.2.3` → `v1.2.4` | rien ne casse |
| ajout compatible | `v1.2.3` → `v1.3.0` | rien ne casse |
| rupture de compatibilité | `v1.2.3` → `v2.0.0` | le code appelant peut casser |

La spécificité Go — et elle surprend tout le monde : **à partir de la version majeure 2, le
numéro majeur fait partie du chemin d'import.**

```go
import "github.com/user/lib"      // v0 ou v1
import "github.com/user/lib/v2"   // v2 et au-delà
```

Le `go.mod` du module doit alors déclarer `module github.com/user/lib/v2`.

La conséquence est puissante : **deux versions majeures peuvent coexister** dans le même
programme. Une migration progressive est possible, dépendance par dépendance — là où d'autres
écosystèmes imposent un basculement global. Le prix est une gymnastique de chemins qui
déroute au début.

Les versions `v0.x.y` sont explicitement **hors garantie** : tout peut casser à chaque
version mineure. Publier en `v0` est une façon de dire « API pas encore stable ».

### MVS : la sélection de version minimale

C'est **la** particularité de Go face à npm, Cargo ou Maven.

Quand plusieurs dépendances exigent des versions différentes d'un même module, Go choisit la
**plus élevée des versions minimales demandées** — pas la dernière disponible.

```
mon module   → requiert  lib v1.2.0
dépendance A → requiert  lib v1.4.0
dépendance B → requiert  lib v1.3.0
                          ────────
Go choisit :             lib v1.4.0     (le maximum des minimums)
```

Conséquences pratiques :

- **Les builds sont reproductibles sans fichier de verrouillage.** Le même `go.mod` produit
  le même graphe de dépendances aujourd'hui et dans cinq ans. Il n'existe pas de
  `package-lock.json` en Go, et ce n'est pas un oubli.
- **Rien ne se met à jour tout seul.** Publier `lib v1.5.0` ne change rien pour personne tant
  que quelqu'un ne demande pas explicitement la mise à jour. C'est le contraire de la
  résolution « toujours le plus récent compatible » de npm.
- **Les mises à jour sont un acte délibéré**, et donc traçable dans un commit.

### Les commandes du quotidien

```bash
go mod init github.com/user/projet   # créer le module
go mod tidy                          # LA commande à connaître : ajoute ce qui manque,
                                     # retire ce qui ne sert plus, met à jour go.sum
go get exemple.com/pkg@v1.2.3        # version précise
go get exemple.com/pkg@latest        # dernière version
go get -u ./...                      # met à jour les dépendances (mineures et correctifs)
go get -u=patch ./...                # correctifs uniquement : plus prudent
go get exemple.com/pkg@none          # retirer une dépendance

go list -m all                       # graphe de modules effectif
go mod why exemple.com/pkg           # POURQUOI cette dépendance est-elle là ?
go mod graph                         # graphe complet, brut
go mod verify                        # vérifie l'intégrité du cache
go mod download                      # pré-télécharge (utile en CI et Docker)
```

`go mod tidy` est à lancer avant chaque commit, comme `gofmt` et `go vet`. Il synchronise
`go.mod` avec la réalité des imports.

Depuis **Go 1.27**, `go mod tidy` fusionne automatiquement les blocs `require` en double pour
les modules déclarant `go 1.27` ou plus — un détail cosmétique qui évite des conflits de
fusion inutiles.

### Installer un outil ≠ ajouter une dépendance

```bash
go install golang.org/x/tools/cmd/goimports@latest   # installe un BINAIRE dans $GOPATH/bin
```

`go install` avec `@version` n'ajoute **rien** au `go.mod` du projet courant. C'est la bonne
façon d'installer un outil.

Pour figer les versions d'outils **partagées par l'équipe**, Go 1.24 a introduit la directive
`tool` dans `go.mod` :

```
tool golang.org/x/tools/cmd/goimports
```
```bash
go tool goimports -w .
```

Cela remplace l'ancien bricolage du fichier `tools.go` avec des imports anonymes, qu'on croise
encore dans beaucoup de projets.

### `go.work` : plusieurs modules en développement

Quand on travaille simultanément sur deux modules qui se référencent :

```bash
go work init ./api ./lib
```

`go.work` redirige localement sans toucher aux `go.mod`. **Ne pas le committer**
(il est dans notre `.gitignore`). C'est le remplaçant propre du `replace` local.

### Publier un module

1. Le code est sur un dépôt public accessible.
2. Le chemin du module correspond à l'URL du dépôt.
3. Créer une étiquette git : `git tag v1.0.0 && git push origin v1.0.0`.
4. C'est tout — il n'y a **pas de registre central** où publier. Le premier `go get` déclenche
   la mise en cache par le proxy.

Point crucial : **une version publiée est immuable et définitivement mise en cache**. Supprimer
l'étiquette ou forcer une réécriture ne la fait pas disparaître du proxy. D'où l'existence de
`retract`, et d'où l'importance de ne pas publier `v1.0.0` à la légère.

## Exemple

```bash
# Création
mkdir biblio && cd biblio
go mod init github.com/willbrid/biblio

# Ajout d'une dépendance
go get github.com/google/uuid@v1.6.0
```

```go
// main.go
package main

import (
	"fmt"

	"github.com/google/uuid"
)

func main() {
	fmt.Println("emprunt", uuid.NewString())
}
```

```bash
$ go mod tidy
$ cat go.mod
module github.com/willbrid/biblio

go 1.27

require github.com/google/uuid v1.6.0

$ cat go.sum
github.com/google/uuid v1.6.0 h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
github.com/google/uuid v1.6.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=

$ go mod why github.com/google/uuid
# github.com/google/uuid
github.com/willbrid/biblio
github.com/google/uuid

$ go list -m all
github.com/willbrid/biblio
github.com/google/uuid v1.6.0
```

Les deux lignes de `go.sum` sont normales : la première est l'empreinte du **contenu** du
module, la seconde celle de son **`go.mod`** — Go doit pouvoir lire le `go.mod` d'une
dépendance pour construire le graphe, sans télécharger tout le code.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `go mod init <chemin>` | Le chemin doit correspondre à l'URL du dépôt si le module sera publié un jour. Le changer après coup casse tous les imports. |
| `go get …@v1.6.0` | Version explicite. Sans `@`, c'est `@latest`, ce qui rend le résultat dépendant du jour. |
| `require github.com/google/uuid v1.6.0` | Version **minimale** demandée, pas la version figée. MVS choisira au moins celle-ci. |
| Deux lignes dans `go.sum` | Contenu du module **et** de son `go.mod` : deux empreintes distinctes. |
| `go mod why` | Répond à la question « qui a amené cette dépendance ? » — la première commande à lancer devant un `go.mod` qui a grossi. |
| `go list -m all` | Le graphe **effectif** après MVS, à ne pas confondre avec le contenu de `go.mod`. |

## Erreurs fréquentes

1. **Ne pas committer `go.sum`** : la vérification d'intégrité est perdue.
2. **Committer un `replace` local** : le module devient inutilisable par les autres.
3. **Committer `go.work`** : il court-circuite les `go.mod` pour tout le monde.
4. **Croire que `go 1.27` est une version minimale de compilateur.**
5. **Oublier `/v2`** dans le chemin d'import et dans `module` lors d'un passage en v2.
6. **Publier `v1.0.0` trop tôt** : la promesse de compatibilité devient contraignante, et la version est immuable.
7. **`go get -u ./...` sans relire le diff** : mises à jour mineures non maîtrisées d'un coup.
8. **`go install` cru équivalent à une dépendance** : ce n'est pas la même chose.
9. **Oublier `go mod tidy`** : `go.mod` diverge des imports réels.
10. **Configurer `GOPRIVATE` trop tard** : le proxy public tente d'atteindre un dépôt privé, et l'échec est incompréhensible.

## Bonnes pratiques Go

- `go mod tidy` avant chaque commit ; `go.mod` et `go.sum` versionnés.
- Chemin de module = URL du dépôt, décidé **dès le premier jour**.
- Versions explicites lors d'un `go get` ; relire le diff de `go.mod`.
- Rester en `v0.x` tant que l'API n'est pas stable.
- `go.work` pour le développement multi-modules, jamais `replace` committé.
- `GOPRIVATE` configuré pour les dépôts d'entreprise.
- Limiter les dépendances : chaque `require` est une surface de maintenance et de sécurité.
  La bibliothèque standard Go couvre énormément — c'est un trait de culture du langage.
- `go mod why` avant d'ajouter une dépendance qui semble déjà présente.

## Ce que je dois retenir

- Un **module** est l'unité de versionnement ; son nom est le chemin d'import racine.
- `go 1.27` déclare la version du **langage**, pas du compilateur.
- **`go.sum` garantit l'intégrité**, `go.mod` déclare les versions.
- Semver obligatoire, et **`/v2` dans le chemin** à partir de la version majeure 2.
- **MVS** choisit le maximum des minimums : builds reproductibles, aucune mise à jour implicite.
- `go mod tidy` est la commande du quotidien.
- Une version publiée est **immuable** ; seul `retract` permet de la désavouer.

➡️ [Exercices](exercices.md)
