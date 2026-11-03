# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-12/lecon-02/`, en partant du projet `svc` de l'exemple.

## Exercices faciles

**E1 — D'autres plateformes.**
Construire `svc` pour `linux/riscv64`, `freebsd/amd64` et `linux/386`. Que dit `file` de chaque binaire ?
Comparer leurs tailles à celle du binaire `linux/amd64` : l'ordre est-il celui que tu attendais ? Construire
aussi pour `js/wasm` : que produit-on, et où ce fichier peut-il s'exécuter ?

**E2 — Les pièges de `-X`.**
Créer un paquet `internal/meta` avec une variable initialisée par une constante, une variable initialisée par
`strings.ToUpper("…")`, et une constante. Tenter de remplacer les trois avec `-X`, puis une variable dont le
nom est mal écrit. Lesquelles changent ? Quels messages donne `go build` ? Proposer une vérification
automatique, dans `build.sh`, qui fait échouer la publication si la version n'a pas été injectée.

**E3 — Code par plateforme.**
Écrire `notify_linux.go` (affiche « notification Linux ») et `notify_other.go` (`//go:build !linux`, affiche
« notification générique »), avec la même fonction `notify()`. Vérifier avec une construction pour Windows.
Puis ajouter un test `slow_test.go` derrière `//go:build integration`. Comment vérifier qu'il n'est pas lancé
par `go test ./...` et qu'il l'est avec `-tags integration` ?

**E4 — Ce que `-s -w` retire.**
Comparer `go tool nm` sur le binaire normal et sur le binaire réduit. Provoquer une panique dans chacun :
la trace de pile change-t-elle ? Si `dlv` est installé (`go install github.com/go-delve/delve/cmd/dlv@latest`),
tenter `dlv exec` sur chacun et poser un point d'arrêt sur `main.main`.

**E5 — Casser la reproductibilité.**
À partir de `build.sh`, trouver **quatre** modifications qui font changer l'empreinte du binaire Linux
entre deux constructions à partir des mêmes sources (par exemple l'heure de build, un autre répertoire sans
`-trimpath`…). Pour chacune, dire si elle est légitime (le binaire **doit** changer) ou un défaut.

---

## Exercice intermédiaire — Un outil de publication en Go

Remplacer `build.sh` par un programme Go `cmd/release` (lancé par `go run ./cmd/release v1.2.0`) qui :

1. Construit les cibles d'une liste (`[]struct{ OS, Arch string }`) en parallèle, au plus 4 à la fois
   (`errgroup.SetLimit`, N6 L9), avec `os/exec` (leçon 1, défi) et les options du cours.
2. Écrit `SHA256SUMS` (`crypto/sha256`), trié par nom de fichier.
3. **Vérifie** chaque binaire produit sans l'exécuter, avec `debug/buildinfo.ReadFile`. Ce paquet lit les
   informations de construction d'un binaire Go de n'importe quelle plateforme. Il faut vérifier que les
   réglages contiennent bien `-trimpath`, `CGO_ENABLED=0`, le bon `GOOS`/`GOARCH`, et que les `-ldflags`
   contiennent `main.version=v1.2.0`. Si l'une de ces vérifications se révèle impossible, expliquer pourquoi
   et en proposer une autre.
4. Refuse de publier si l'arbre Git est modifié (`+dirty`, leçon 1) ou si la version ne commence pas
   par `v`.
5. Teste la partie vérification sur un binaire construit dans `t.TempDir()`.

---

## Défi — Le même binaire, ailleurs

1. Construire `svc` avec `build.sh` sur ta machine, puis **dans un conteneur** de l'image officielle
   `golang:1.27.3`, en montant le projet :
   `docker run --rm -v "$PWD":/src -w /src golang:1.27.3 ./build.sh v1.2.0`.
   Les sommes de contrôle sont-elles identiques ?
2. Recommencer en retirant `-trimpath` du script. Puis avec `-trimpath`, mais `CGO_ENABLED=1` pour la cible
   `linux/amd64`.
3. Recommencer en changeant **seulement** la version de Go de l'image (`golang:1.26`). Que dit `go` ? Que
   change la ligne `toolchain` dans `go.mod` ?
4. Rédiger en dix lignes la procédure qui permettrait à un utilisateur de vérifier, seul, qu'un binaire
   publié correspond à l'étiquette `v1.2.0` du dépôt.

---

## Quiz

1. Pourquoi un binaire Go se déploie-t-il facilement ? Quand dépend-il de la libc ?
2. Que faut-il pour cross-compiler un programme Go pur ?
3. Que fait `CGO_ENABLED=0` ?
4. Deux façons de réserver un fichier à une plateforme ?
5. Comment ne lancer certains tests qu'à la demande ?
6. Quelles variables `-X` peut-il remplacer ? Que se passe-t-il dans les autres cas ?
7. Que retirent `-s` et `-w` ? Que garde-t-on ?
8. Qu'est-ce qu'un build reproductible, et à quoi sert-il ?
9. Que fait `-trimpath` ?
10. Pourquoi ne pas inscrire l'heure du build dans le binaire ?
