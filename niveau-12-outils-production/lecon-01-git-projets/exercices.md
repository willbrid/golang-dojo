# Leçon 1 — Exercices

Travailler dans un dépôt de test créé pour l'occasion (`mktemp -d`), jamais dans un vrai dépôt partagé.

## Exercices faciles

**E1 — Le binaire qui salit.**
Dans le dépôt de l'exemple, supprimer le répertoire `bin/` et la ligne `/bin/` du `.gitignore` (et valider), puis construire avec
`go build -o app ./cmd/app` à la racine **deux fois de suite**. Quelle version affiche chaque binaire ?
Pourquoi le premier build est-il propre et le second non ?

**E2 — Lire un binaire de l'extérieur.**
Sur le binaire de l'étape 4, lancer `go version -m bin/app`. Retrouver la version, le commit, la date et
l'état du dépôt. Faire de même sur un outil installé par `go install` (par exemple `staticcheck`, N5 L8) :
quelle version et quelles dépendances contient-il ?

**E3 — Ordonner des versions.**
Ranger dans l'ordre croissant : `v1.2.0`, `v1.10.0`, `v1.2.1-0.20260101000000-abcdefabcdef`, `v1.2.0-rc.2`,
`v1.2.0-rc.10`, `v1.3.0-beta`, `v0.9.9`. Vérifier avec `golang.org/x/mod/semver` si elle est dans le
cache (`semver.Sort`), sinon en écrivant un petit test qui compare les chaînes selon les règles de semver.
Pourquoi `rc.10` passe-t-il après `rc.2` ?

**E4 — Le tag déplacé.**
Expliquer, sans le faire sur un vrai module public, ce qui arriverait si l'on supprimait l'étiquette
`v1.0.0` d'un module déjà téléchargé par d'autres, puis si on la recréait sur un autre commit. Qui verrait
quoi ? Que faire à la place ?

**E5 — Un `bisect` qui doit sauter.**
Modifier le script pour que le commit « étape 2 » ne compile plus (une erreur de syntaxe), et que l'« étape 4 »
corrige la syntaxe, la régression restant à l'étape 3. Que conclut `git bisect run go test ./internal/price/` ?
Écrire un petit script `test.sh` qui renvoie 125 quand `go build ./...` échoue, et relancer avec
`git bisect run ./test.sh`.

---

## Exercice intermédiaire — Une commande `version`

Pour le service de prêts du niveau 10 (ou ton projet 7) :

1. Écrire un paquet `internal/buildinfo` avec `func Read() Info`. `Info` contient la version, le commit
   (12 caractères), la date du commit, `Dirty bool` et la version de Go (`runtime.Version()`).
2. Le service affiche ces informations au démarrage et répond à `GET /version` en JSON, sur l'adresse
   d'administration (N10 L10).
3. Tester `Read` sans construire de binaire (quelle partie peut-on tester, et comment rendre le reste
   injectable ?).
4. Ajouter une option `-version` à la commande, qui affiche la version et quitte.
5. Que doit afficher le service quand les informations manquent (`go run`, `-buildvcs=false`) ?

---

## Défi — Des notes de version automatiques

1. Écrire un programme Go `relnotes` qui lit `git log --format=…` entre deux étiquettes, analyse les
   messages selon la convention de la leçon, et produit des notes de version en Markdown groupées par type
   (fonctionnalités, corrections, ruptures).

   Lancer un programme externe est nouveau. Le paquet `os/exec` le fait :
   - `cmd := exec.CommandContext(ctx, "git", "log", …)` prépare la commande, sans passer par un shell (les
     arguments ne sont pas interprétés) ;
   - `cmd.Dir` fixe son répertoire de travail ;
   - `out, err := cmd.Output()` la lance et renvoie sa sortie standard ; `err` n'est pas nul si le code de
     retour n'est pas 0.

   Choisir un format (`--format=%H%x1f%s%x1f%b%x1e`, par exemple) que les messages ne peuvent pas
   contenir, pour pouvoir découper la sortie sans ambiguïté.
2. Il propose aussi la **prochaine version** : majeure s'il y a une rupture, mineure s'il y a un `feat`,
   corrective sinon. Attention aux versions `v0.x.y`, où une rupture n'incrémente que la version mineure.
3. Tester l'analyse des messages par une table de tests (sans Git), et le programme complet sur un dépôt
   créé dans `t.TempDir()`, avec des dates fixées.
4. Que faire des messages qui ne suivent pas la convention ? Les ignorer, les signaler, faire échouer ?
   Justifier.

---

## Quiz

1. Quels fichiers d'un module valide-t-on toujours ? Lesquels ignore-t-on ?
2. Que signifie « développement sur le tronc » ?
3. Que doit contenir le corps d'un message de commit ?
4. Comment annoncer une rupture de compatibilité dans un message de commit ?
5. Pourquoi ne faut-il jamais déplacer une étiquette de version publiée ?
6. Quelle version `go build` inscrit-il un commit après `v1.4.2` ?
7. Que signifie `+dirty`, et quel fichier en est souvent la cause ?
8. Que vaut la version d'un programme lancé par `go run` ?
9. Que fait `git bisect run`, et quel code de retour sert à sauter un commit ?
10. Comment étiqueter la version d'un module situé dans un sous-répertoire du dépôt ?
