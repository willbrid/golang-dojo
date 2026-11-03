# Leçon 8 — Sécurité et secrets

> Un service en production est exposé à trois sortes de risques.
> - **Son propre code** : les failles traitées au niveau 7 (injection, SSRF, authentification).
> - **Le code des autres** : des dépendances vulnérables, une chaîne d'approvisionnement compromise.
> - **Ses secrets** : mots de passe, jetons et clés, qui fuient par un dépôt, une image ou un journal.
>
> Cette leçon traite les deux dernières, avec les outils de l'écosystème Go : `govulncheck`, les sommes
> de contrôle des modules, la gestion du cycle de vie d'un secret. Elle se termine par la liste de
> durcissement d'un service, qui rassemble tout le niveau.

## Objectifs

- Trouver les vulnérabilités **réellement atteignables** dans le code et dans un binaire avec `govulncheck`.
- Comprendre ce qui protège la chaîne d'approvisionnement des modules Go, et ce qui ne la protège pas.
- Gérer un secret sur tout son cycle de vie : création, stockage, distribution, rotation, révocation.
- Savoir pourquoi un secret validé dans Git est compromis, même après suppression.
- Vérifier un jeton en temps constant, et mesurer pourquoi.
- Réduire la surface d'attaque d'un service, point par point.

## Explication

### Les vulnérabilités des dépendances : `govulncheck`

`govulncheck` (présenté en N5 L8) confronte les dépendances d'un module à la base de vulnérabilités de Go
(vuln.go.dev). Sa particularité est l'**analyse d'atteignabilité** : il ne se contente pas de dire « ce
module a une vulnérabilité », il cherche si **ton code appelle** la fonction vulnérable.

L'exemple le mesure avec une ancienne version de `golang.org/x/text` (v0.3.5), qui a plusieurs
vulnérabilités connues :

| Situation | Résultat | Code de sortie |
|---|---|---|
| le paquet `language` est importé, `language.Parse` n'est pas appelée | « Your code is affected by 0 vulnerabilities », avec 2 vulnérabilités dans les paquets importés et 2 dans les modules requis, signalées pour information | 0 |
| `language.Parse` est appelée sur une entrée de l'utilisateur | GO-2021-0113 (lecture hors limites), avec la **trace d'appel** : `svc.main calls language.Parse` | **3** |
| le **binaire** construit (`-mode binary`) | la même vulnérabilité, trouvée sans le code source | 3 |
| après `go get golang.org/x/text@latest` (v0.43.0) | « No vulnerabilities found » | 0 |

L'analyse d'atteignabilité réduit le bruit : on corrige d'abord ce qui est appelé. Mais une vulnérabilité
« non appelée » aujourd'hui peut le devenir au prochain commit. On met donc à jour **régulièrement**
(Dependabot, leçon 4), pas seulement quand `govulncheck` échoue.

En CI, `govulncheck ./...` échoue (code 3) si une vulnérabilité est atteignable (leçon 4). Le mode binaire sert
à vérifier un binaire **déjà publié** quand une nouvelle vulnérabilité est annoncée : faut-il republier ?

La bibliothèque standard est concernée aussi. Une vulnérabilité de `net/http` se corrige en changeant de
version de Go (`toolchain` dans `go.mod`, leçon 2). C'est une raison de suivre les versions correctives
(1.27.x).

### La chaîne d'approvisionnement des modules

Ce que l'outil `go` garantit :

- **`go.sum`** contient l'empreinte de chaque version de module utilisée. Un module dont le contenu ne
  correspond pas est refusé (N2 L9). `go mod verify` revérifie le cache local.
- **La base de sommes de contrôle** (`sum.golang.org`) est un journal public, en ajout seul, des empreintes
  de toutes les versions publiques. Le premier qui télécharge une version fixe son empreinte **pour tout le
  monde**. Un auteur, ou un attaquant qui prend son compte, ne peut pas servir un contenu différent à
  certains utilisateurs sans être détecté (leçon 1, le tag déplacé).
- **`go build` ne modifie jamais `go.mod`** (mode `-mod=readonly` par défaut). Une dépendance ne s'ajoute pas
  « toute seule » pendant un build.
- **Le téléchargement d'une version de Go** (`GOTOOLCHAIN`) est vérifié de la même façon.

Ce que l'outil `go` ne garantit **pas** :

- **qu'une dépendance est honnête.** La base garantit que tout le monde reçoit le **même** code, pas que ce
  code est sain. Une version malveillante publiée par son auteur sera la même pour tous.
- **qu'un nom est le bon.** Un module au nom presque identique (`github.com/sirupsen/logrus` contre une
  imitation) est un autre module, parfaitement vérifié.

D'où les règles :

- **peu de dépendances.** Chaque dépendance est du code exécuté avec les droits du service. La bibliothèque
  standard de Go couvre déjà HTTP, JSON, crypto et SQL ;
- **une revue de chaque nouvelle dépendance** : qui la maintient, depuis quand, avec quelles dépendances à
  elle (`go mod graph`), et pour quoi faire (`go mod why -m <module>`) ;
- **des modules privés hors de la base publique.** `GOPRIVATE=github.com/monentreprise/*` : ni le proxy
  public ni `sum.golang.org` ne sont interrogés pour eux. Leur nom ne fuit pas, et ils sont téléchargés
  directement, avec les identifiants Git.

### Le cycle de vie d'un secret

| Étape | Bonne pratique |
|---|---|
| **création** | aléatoire et long : `crypto/rand` (`rand.Text()` : 128 bits), jamais `math/rand` ni un mot choisi par un humain |
| **stockage** | un gestionnaire de secrets (celui de l'hébergeur, ou un outil dédié), chiffré, avec des accès nominatifs et journalisés ; jamais Git, jamais une image (leçon 3), jamais un fichier de configuration validé (leçon 5) |
| **distribution** | un fichier monté au démarrage (`*_FILE`, leçon 5), ou une variable d'environnement à défaut |
| **usage** | lu une fois, jamais journalisé (leçon 6), jamais renvoyé dans une erreur |
| **rotation** | périodique, et **sans interruption** : le service accepte **deux** valeurs pendant la transition (l'ancienne et la nouvelle), puis on retire l'ancienne |
| **révocation** | immédiate en cas de doute ; on suppose compromis tout secret qui a pu être vu |

La rotation demande que le code sache vivre avec deux valeurs (un `Verifier` par jeton valide), et que la
configuration se recharge (leçon 5, défi) ou que le service redémarre sans interruption (leçon 9). Un secret
qu'on ne sait pas changer sans arrêter le service ne sera jamais changé.

### Un secret dans Git est compromis

Supprimer un fichier `.env` dans un commit ne le retire pas de l'**historique**. L'exemple le montre :
`git log -p -S` et `git grep` sur toutes les révisions retrouvent le mot de passe, alors qu'il n'est plus suivi
dans le dernier commit. Le dépôt a pu être cloné, mis en cache, sauvegardé, indexé.

La seule réponse sûre est la **révocation** : on change le secret, immédiatement. Réécrire l'historique
peut être utile ensuite, pour ne pas laisser traîner l'ancien, mais cela ne rend pas secret ce qui a été
publié.

La prévention : un `.gitignore` dès le premier commit (leçon 1), et un **analyseur de secrets** en CI et
avant chaque commit. Il en existe plusieurs, libres. Ils reconnaissent les formats de clés connus et les
chaînes à forte entropie.

### Vérifier un jeton en temps constant

N7 L8 l'a posé comme règle ; l'exemple le mesure. Pour un jeton attendu de 4 096 octets :

| Comparaison | Différence au premier octet | Différence au dernier octet |
|---|---|---|
| `got == want` | 3,9 ns | 65 ns |
| `subtle.ConstantTimeCompare` sur les empreintes SHA-256 | 12,7 µs | 12,7 µs |

La comparaison ordinaire s'arrête au premier octet différent. Son temps révèle **combien d'octets du début
sont corrects**. En mesurant beaucoup de requêtes, un attaquant devine le jeton octet par octet. La
comparaison en temps constant prend le même temps quelle que soit la position de la différence.

Comparer les **empreintes** (SHA-256) plutôt que les jetons a deux avantages :

- les deux valeurs comparées ont toujours la même longueur (32 octets), ce qui évite de révéler la
  longueur du jeton attendu ;
- le service ne garde en mémoire que l'empreinte.

Le coût (12,7 µs ici) vient du hachage de 4 Kio. Pour un jeton réaliste de 26 caractères, il est négligeable.

### Réduire la surface d'attaque

La surface d'attaque est tout ce qu'un attaquant peut atteindre. La liste de durcissement d'un service Go,
qui rassemble le niveau :

| Point | Leçon |
|---|---|
| délais (`ReadHeaderTimeout`…) et limites de taille sur le serveur HTTP | N7 L9 |
| `/debug/pprof` et `/metrics` sur l'adresse d'administration seulement | N10 L10, N11 L4, leçon 7 |
| image *distroless* `nonroot`, système de fichiers en lecture seule, aucune capacité | leçon 3 |
| binaire statique, `-trimpath` (pas de chemins de la machine de build) | leçon 2 |
| ports publiés sur `127.0.0.1` en local ; base de données jamais exposée | leçon 3 |
| configuration validée au démarrage ; secrets par fichier | leçon 5 |
| aucun secret ni donnée personnelle dans les journaux | leçon 6 |
| `govulncheck` en CI, dépendances à jour, versions correctives de Go | cette leçon, leçon 4 |
| CI aux permissions minimales, actions épinglées | leçon 4 |
| TLS pour tout trafic qui sort de la machine | N7 L11 |
| requêtes SQL paramétrées | N8 L2 |
| URL sortantes validées (SSRF) | N7 L9 |

### Signer ce qu'on publie

`SHA256SUMS` (leçon 2) prouve qu'un fichier n'a pas été modifié **depuis** la publication du fichier de
sommes, mais pas **qui** l'a publié. Une **signature** le prouve. L'écosystème des conteneurs utilise
couramment Sigstore (`cosign`) en mode « sans clé ». En CI, le job reçoit une identité temporaire de GitHub
(`permissions: id-token: write`, leçon 4), signe l'image ou les sommes, et la signature est inscrite dans un
journal public. L'utilisateur vérifie que l'artefact a été signé par **ce** workflow, de **ce** dépôt.

Cette partie n'a pas été exécutée pour le cours : elle demande une vraie CI et un vrai registre. L'exercice
défi la met en œuvre dans ton dépôt de test.

## Exemple

Trois démonstrations : une dépendance vulnérable vue par `govulncheck`, un secret retrouvé dans
l'historique Git, et la vérification d'un jeton en temps constant.

**1. `govulncheck`.** Un programme qui importe `golang.org/x/text@v0.3.5` sans appeler `language.Parse` :

```go
package main

import (
	"fmt"

	"golang.org/x/text/language"
)

func main() {
	fmt.Println(language.English) // le paquet est importé, mais language.Parse n'est pas appelée
}
```

```
$ go run golang.org/x/vuln/cmd/govulncheck@v1.6.0 ./...
=== Symbol Results ===

No vulnerabilities found.

Your code is affected by 0 vulnerabilities.
This scan also found 2 vulnerabilities in packages you import and 2
vulnerabilities in modules you require, but your code doesn't appear to call
these vulnerabilities.
Use '-show verbose' for more details.
```

La même chose, en appelant `language.Parse` sur une entrée de l'utilisateur :

```go
func main() {
	// Une étiquette de langue venue de l'utilisateur (l'en-tête Accept-Language, par exemple).
	tag, err := language.Parse(os.Args[1])
	fmt.Println(tag, err)
}
```

```
$ go run golang.org/x/vuln/cmd/govulncheck@v1.6.0 ./...
=== Symbol Results ===

Vulnerability #1: GO-2021-0113
    Out-of-bounds read in golang.org/x/text/language
  More info: https://pkg.go.dev/vuln/GO-2021-0113
  Module: golang.org/x/text
    Found in: golang.org/x/text@v0.3.5
    Fixed in: golang.org/x/text@v0.3.7
    Example traces found:
      #1: cmd/svc/main.go:12:28: svc.main calls language.Parse

Your code is affected by 1 vulnerability from 1 module.
[…]
exit status 3
$ go build -o bin/svc ./cmd/svc && go run golang.org/x/vuln/cmd/govulncheck@v1.6.0 -mode binary bin/svc
Vulnerability #1: GO-2021-0113
Your code is affected by 1 vulnerability from 1 module.
$ go get golang.org/x/text@latest && go run golang.org/x/vuln/cmd/govulncheck@v1.6.0 ./...
go: upgraded golang.org/x/text v0.3.5 => v0.43.0
No vulnerabilities found.
$ go mod verify
all modules verified
```

(`-show verbose` détaille les vulnérabilités non atteignables, par exemple GO-2022-1059, un déni de service
par un en-tête `Accept-Language` forgé, corrigé en v0.3.8.)

**2. Le secret dans l'historique.**

```bash
git init -q -b main
echo "# svc" > README.md
printf 'DATABASE_URL=postgres://app:Pr0d-s3cr3t@db.interne/app\n' > .env
git add . && git commit -qm "feat: première version"     # l'erreur
git rm -q --cached .env && echo ".env" > .gitignore       # la « correction »
git add .gitignore && git commit -qm "chore: ne plus suivre .env"
```

```
$ git ls-files
.gitignore
README.md
$ git log -p --all -S 'Pr0d-s3cr3t' --format='%h %s' | grep -E '^[0-9a-f]{7} |Pr0d'
b62d0ad chore: ne plus suivre .env
-DATABASE_URL=postgres://app:Pr0d-s3cr3t@db.interne/app
e783688 feat: première version
+DATABASE_URL=postgres://app:Pr0d-s3cr3t@db.interne/app
$ git grep -n 'Pr0d-s3cr3t' $(git rev-list --all)
e7836885cf624eb30c65ce1c0cc551e41515abf2:.env:1:DATABASE_URL=postgres://app:Pr0d-s3cr3t@db.interne/app
```

**3. Le jeton.**

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27
```

```go
// Fichier : internal/auth/auth.go
// Package auth : vérifier un jeton d'API sans fuite par le temps, et en générer un.
package auth

import (
	"crypto/rand"
	"crypto/sha256"
	"crypto/subtle"
)

// NewToken renvoie un jeton aléatoire, imprévisible (crypto/rand, jamais math/rand).
func NewToken() string { return rand.Text() } // 26 caractères, 128 bits d'aléa

// Verifier ne garde que l'empreinte du jeton attendu : une fuite de la mémoire ou d'une sauvegarde
// ne donne pas le jeton lui-même.
type Verifier struct{ want [32]byte }

func NewVerifier(token string) Verifier { return Verifier{sha256.Sum256([]byte(token))} }

// Check compare en temps constant : la durée ne dépend pas du nombre de caractères corrects.
func (v Verifier) Check(got string) bool {
	h := sha256.Sum256([]byte(got))
	return subtle.ConstantTimeCompare(h[:], v.want[:]) == 1
}

// CheckNaive : la comparaison ordinaire s'arrête au premier octet différent.
func CheckNaive(got, want string) bool { return got == want }
```

```go
// Fichier : internal/auth/auth_test.go
package auth

import (
	"strings"
	"testing"
)

func TestCheck(t *testing.T) {
	tok := NewToken()
	v := NewVerifier(tok)
	if !v.Check(tok) || v.Check(tok+"x") || v.Check("") {
		t.Fatal("vérification incorrecte")
	}
	if len(tok) != 26 || NewToken() == tok {
		t.Fatalf("jeton : %q", tok)
	}
}

var sink bool

// Le temps de la comparaison naïve révèle combien d'octets du début sont corrects.
func BenchmarkCompare(b *testing.B) {
	want := strings.Repeat("a", 4096)
	early := "b" + want[1:]          // différent dès le premier octet
	late := want[:len(want)-1] + "b" // différent au dernier octet
	v := NewVerifier(want)
	for _, tc := range []struct{ name, got string }{{"premier-octet", early}, {"dernier-octet", late}} {
		b.Run("naïve/"+tc.name, func(b *testing.B) {
			for b.Loop() {
				sink = CheckNaive(tc.got, want)
			}
		})
		b.Run("constant/"+tc.name, func(b *testing.B) {
			for b.Loop() {
				sink = v.Check(tc.got)
			}
		})
	}
}
```

```
$ go test -run XXX -bench Compare -count 5 ./internal/auth/      # médianes
BenchmarkCompare/naïve/premier-octet-8       3.86 ns/op
BenchmarkCompare/naïve/dernier-octet-8      65.3  ns/op
BenchmarkCompare/constant/premier-octet-8   12689 ns/op
BenchmarkCompare/constant/dernier-octet-8   12728 ns/op
```

## Explication du code

| Élément | Rôle |
|---|---|
| `rand.Text()` (`crypto/rand`) | Un jeton de 26 caractères en base 32 : 128 bits d'aléa, imprévisible. `math/rand` est fait pour la simulation, et une graine connue (ou devinée) redonne la même suite. |
| `Verifier{want [32]byte}` | Seule l'empreinte du jeton est gardée. |
| `sha256.Sum256` des deux côtés | Deux valeurs de 32 octets : la comparaison ne dépend pas de la longueur du jeton proposé. |
| `subtle.ConstantTimeCompare(…) == 1` | Parcourt tous les octets, sans branche dépendant des données. Elle renvoie 1 si égales. |
| `CheckNaive` | `==` sur des `string` : le moyen le plus court de comparer, et le plus bavard sur le temps. |
| `early` / `late` dans le benchmark | Même longueur, une différence au début ou à la fin : seule la position change, et la comparaison naïve passe de 3,9 à 65 ns. |
| `-mode binary` | `govulncheck` lit les informations de construction du binaire (leçon 2) : versions des modules et symboles présents. |
| `git log -p -S chaîne` | Les commits qui ajoutent ou retirent une chaîne, avec leurs diffs. |

## Erreurs fréquentes

1. **Ignorer `govulncheck`** parce que « la vulnérabilité n'est pas appelée », et ne jamais mettre à jour.
2. **Ajouter une dépendance pour une fonction de dix lignes**, sans regarder qui la maintient.
3. **Croire que `go.sum` garantit qu'une dépendance est saine** : il garantit seulement qu'elle est la même
   pour tous.
4. **Un secret validé dans Git**, « corrigé » par un commit de suppression, sans révocation.
5. **Un secret qu'on ne sait pas changer** sans arrêter le service.
6. **`math/rand` pour un jeton**, un identifiant de session ou un sel.
7. **`==` pour comparer un jeton** ou une signature.
8. **Des modules privés interrogés sur le proxy public** : leurs noms fuient, et le téléchargement échoue.
9. **Ne jamais changer de version corrective de Go** : les failles de la bibliothèque standard restent.

## Bonnes pratiques Go

- `govulncheck ./...` en CI ; `-mode binary` sur les binaires publiés quand une alerte tombe.
- Dependabot (ou équivalent) pour `gomod`, `github-actions` et les images de base ; versions correctives de
  Go suivies (`toolchain`).
- Peu de dépendances, chacune revue (`go mod why`, `go mod graph`) ; `GOPRIVATE` pour les modules privés.
- Secrets créés par `crypto/rand`, stockés dans un gestionnaire de secrets, montés en fichier, jamais
  journalisés, et changeables sans interruption.
- Un analyseur de secrets en CI ; en cas de fuite : révoquer d'abord, nettoyer ensuite.
- `subtle.ConstantTimeCompare` (ou `hmac.Equal`) sur des empreintes pour tout secret comparé.
- La liste de durcissement relue à chaque nouveau service.

## Ce que je dois retenir

- `govulncheck` dit si **ton code appelle** une fonction vulnérable ; code de sortie 3 sinon.
- `go.sum` et `sum.golang.org` garantissent que tout le monde reçoit le même code, pas qu'il est sain.
- Un secret a un cycle de vie ; un secret qu'on ne peut pas changer sans arrêt finit par fuir pour de bon.
- Un secret validé dans Git reste dans l'historique : il est compromis, on le révoque.
- `==` sur un jeton fuit par le temps (3,9 ns contre 65 ns selon la position de l'erreur).

---

➡️ [Exercices](exercices.md)
