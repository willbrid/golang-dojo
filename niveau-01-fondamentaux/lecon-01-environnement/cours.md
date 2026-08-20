# Leçon 1 — L'environnement Go, la structure d'un programme, la chaîne d'outils

## Objectifs

1. Expliquer ce qu'est un **module**, un **package**, et pourquoi `package main` est spécial.
2. Créer un projet Go à partir de rien avec `go mod init`.
3. Utiliser `go run`, `go build`, `go fmt`, `go vet`, `go test` et savoir **à quoi sert chacun**.
4. Lire et écrire un programme Go minimal sans copier-coller.
5. Comprendre pourquoi Go compile vers un **binaire unique** et ce que ça change concrètement.

## Explication

### Ce qu'est Go, en une phrase honnête

Go est un langage **compilé**, **statiquement typé**, à **ramasse-miettes** (garbage
collector), conçu chez Google en 2007 pour un problème très concret : des programmes
serveurs écrits par de grandes équipes, qui compilent lentement et deviennent illisibles.
La réponse de Go a été le **minimalisme délibéré** : 25 mots-clés, pas d'héritage de
classes, pas d'exceptions, une seule façon de formater le code.

Traduction pour un débutant : **Go est petit**. On peut réellement connaître *tout* le
langage. C'est rare et précieux. En contrepartie, Go fait parfois écrire trois lignes là
où Python en écrit une — c'est un choix assumé, pas un oubli.

> **Notions générales, si elles ne sont pas acquises :**
>
> *Compilé* signifie qu'un programme (le compilateur) traduit le texte source en
> instructions machine **avant** l'exécution, dans un fichier exécutable. Python ou
> JavaScript sont *interprétés* : un programme lit le code et l'exécute au vol, il doit
> donc être installé sur la machine cible. Conséquence pratique énorme : un binaire Go
> compilé se copie sur un serveur nu et fonctionne — **aucun runtime, aucune dépendance à
> installer**. C'est la raison n°1 pour laquelle Go domine dans les conteneurs et le
> DevOps : Docker, Kubernetes et Terraform sont écrits en Go.
>
> *Statiquement typé* signifie que le type de chaque variable est connu et vérifié à la
> compilation. Une faute de type devient une erreur de compilation, pas un plantage à 3h
> du matin en production.

### Les trois unités d'organisation : module, package, fichier

C'est **le** point que les débutants confondent le plus. Fixons-le tout de suite :

```
MODULE   = une unité de versionnement et de dépendances. Un dépôt git, en général.
           Défini par un fichier go.mod à la racine. Son nom est le chemin d'import racine.
           Exemple : github.com/willbrid/golang-dojo

  PACKAGE = un répertoire. Tous les fichiers .go d'un même répertoire appartiennent
            au même package. C'est l'unité de compilation et de visibilité.
            Exemple : le package « calc » dans le répertoire calc/

    FICHIER = un simple découpage de confort. Aucune importance sémantique : couper un
              package en 1 ou 5 fichiers ne change strictement rien au programme.
```

Retenir la règle la plus contre-intuitive pour qui vient de Java, C# ou Python :
**en Go, le package est le répertoire, pas le fichier**. Deux fichiers du même répertoire
se voient mutuellement sans aucun import ; ils partagent le même espace de noms.

### `package main` : ce qui rend un programme exécutable

Un package nommé `main` **et** contenant une fonction `func main()` produit un
**exécutable**. Tout autre nom de package produit une **bibliothèque**, qui ne peut pas
être lancée, seulement importée.

## Exemple

```bash
mkdir -p ~/build/golang-dojo/mes-solutions/niveau-01/lecon-01/hello
cd ~/build/golang-dojo/mes-solutions/niveau-01/lecon-01/hello
go mod init exemple.com/hello
```

`main.go` :

```go
// Package main est le point d'entrée d'un programme exécutable.
package main

import (
	"fmt"
	"os"
)

// greeting construit le message de salutation.
// Elle commence par une minuscule : elle n'est visible que dans ce package.
func greeting(name string) string {
	if name == "" {
		name = "monde"
	}
	return fmt.Sprintf("Bonjour, %s !", name)
}

// main est la fonction appelée automatiquement au démarrage du programme.
func main() {
	// os.Args contient les arguments de la ligne de commande.
	// os.Args[0] est le nom du programme lui-même, d'où le découpage à partir de 1.
	args := os.Args[1:]

	name := ""
	if len(args) > 0 {
		name = args[0]
	}

	fmt.Println(greeting(name))
}
```

```bash
go run .            # → Bonjour, monde !
go run . Willbrid   # → Bonjour, Willbrid !
```

## Explication du code

| Élément | Ce qui se passe et **pourquoi** |
|---|---|
| `package main` | Obligatoire en **première ligne de code** de tout fichier `.go`. Déclare à quel package appartient le fichier. `main` = produis un exécutable. |
| `import ( … )` | Bloc d'imports groupés. `"fmt"` = *format*, entrées/sorties formatées ; `"os"` = accès au système. **Go refuse de compiler si un package importé n'est pas utilisé.** Choix volontaire : les imports morts pourrissent les bases de code. |
| `func greeting(name string) string` | Le **nom vient avant le type** (`name string`, pas `string name`). C'est l'inverse du C/Java, et c'est délibéré : ça se lit de gauche à droite. Le `string` final est le type de retour. |
| `fmt.Sprintf` | Construit une chaîne formatée et la **retourne** (le `S` = *String*). `fmt.Printf` l'écrirait sur la sortie standard. `%s` est un verbe de formatage remplacé par la valeur. |
| `func main()` | Signature imposée : **aucun paramètre, aucune valeur de retour**. On ne reçoit pas `argv` en paramètre comme en C — on lit `os.Args`. Le programme s'arrête quand `main` retourne. |
| `args := os.Args[1:]` | `:=` est la **déclaration courte** : elle déclare la variable *et* déduit son type. Utilisable uniquement dans une fonction. `[1:]` est un **slice** : « du 1ᵉʳ élément jusqu'à la fin ». |
| `len(args)` | Fonction intégrée (*builtin*). Pas de méthode `.length` : `len()` est universelle. |
| minuscule vs majuscule | `greeting` → **non exporté**, invisible hors du package. `Println` → **exporté**. En Go, la casse de la première lettre *est* le modificateur de visibilité : il n'existe pas de mot-clé `public`/`private`. |

### La chaîne d'outils

```bash
go run .                  # compile dans un répertoire temporaire ET exécute. Pour itérer vite.
go build                  # compile et produit un binaire dans le répertoire courant.
go build -o bin/hello .   # …à l'emplacement choisi.
go fmt ./...              # reformate selon LE style Go officiel. Non négociable.
go vet ./...              # analyse statique : bugs probables que le compilateur laisse passer.
go test ./...             # exécute les tests (niveau 3).
go doc fmt.Sprintf        # documentation dans le terminal.
go mod tidy               # synchronise go.mod avec les imports réellement utilisés.
```

`./...` signifie « ce répertoire **et tous ses sous-répertoires** ». On le tape des milliers de fois.

**`go fmt`** applique `gofmt`, le formateur officiel, qui n'a **aucune option de style**.
Pas de débat tabulations/espaces (c'est tabulations), pas de débat sur les accolades.
C'est l'une des meilleures décisions de Go : la communauté entière écrit du code
visuellement identique. Configurer l'éditeur pour formater à chaque sauvegarde, et ne
plus jamais réfléchir à la mise en forme.

**`go vet`** est différent du compilateur : il détecte du code *qui compile mais qui est
probablement faux*. Exemple classique : `fmt.Printf("%d", "texte")` compile parfaitement
mais est absurde ; `go vet` le signale. Depuis **Go 1.27**, `go test` lance en plus la
vérification `stdversion` par défaut : elle détecte l'usage d'une API plus récente que la
version Go déclarée dans `go.mod`.

### Le fichier `go.mod`

```
module exemple.com/hello

go 1.27
```

- La ligne `module` définit le **chemin d'import racine**. Convention : l'URL du dépôt
  sans `https://`. Elle n'a pas besoin d'exister réellement tant que le module n'est pas
  publié, mais autant prendre l'habitude.
- La ligne `go 1.27` n'est **pas** « la version qui compile ». C'est la version du
  **langage** demandée : elle active les fonctionnalités de 1.27 et sert de contrat de
  compatibilité. Un Go 1.28 compilera ce module en respectant la sémantique 1.27.

## Erreurs fréquentes

1. **Créer un fichier `.go` sans `go.mod`** → `go: cannot find main module`. Réflexe : toujours `go mod init <chemin>` d'abord.
2. **Importer un package sans l'utiliser** → `"os" imported and not used`. Ce n'est pas un avertissement, c'est une **erreur bloquante**. Idem pour une variable locale déclarée et jamais lue.
3. **Croire que `go run .` produit un binaire.** Non : il compile dans un répertoire temporaire et jette le résultat. C'est `go build` qui produit un fichier.
4. **Mettre l'accolade ouvrante à la ligne suivante :**
   ```go
   func main()
   {          // ← ERREUR DE COMPILATION, pas un choix de style
   ```
   Go insère automatiquement des points-virgules en fin de ligne. `func main()` seul sur sa
   ligne devient `func main();`. L'accolade **doit** rester sur la même ligne. C'est la
   conséquence directe d'une règle du langage, pas une lubie du formateur.
5. **Nommer un répertoire `hello` et le package `salut`.** Autorisé, mais déroutant : par convention le nom du package est celui du répertoire.
6. **Prendre `os.Args[0]` pour le premier argument.** C'est le chemin du programme.

## Bonnes pratiques Go

- **Formater à la sauvegarde**, toujours. Du code non formaté est rejeté en revue.
- Lancer `go vet ./...` avant chaque commit. Coût : deux secondes.
- Noms **courts** et sans redondance : `name` et non `nameString`, `buf` et non
  `theBufferForReading`. Plus la portée d'une variable est courte, plus son nom peut
  l'être. `i` dans une boucle de trois lignes est parfait ; `i` en variable globale est une faute.
- Style Go : `camelCase` pour le non exporté, `PascalCase` pour l'exporté.
  **Jamais de `snake_case`** pour les identifiants Go.
- Un commentaire de documentation commence par le nom de l'élément documenté :
  `// greeting construit …`. C'est ce que `go doc` affiche.

## Ce que je dois retenir

- **Module** (`go.mod`, dépendances) ⊃ **package** (un répertoire, visibilité) ⊃ **fichier** (confort de lecture).
- `package main` + `func main()` = exécutable. Tout le reste = bibliothèque.
- **La casse de la première lettre définit la visibilité.** Majuscule = exporté.
- Un import ou une variable locale inutilisé = **erreur de compilation**, pas un avertissement.
- `gofmt` n'a pas d'options : le style Go n'est pas un sujet de débat.
- Go compile vers un **binaire autonome** : pas de runtime à déployer.
- Séparer **le calcul** de **l'affichage** : c'est la base de la testabilité.

➡️ [Exercices](exercices.md)
