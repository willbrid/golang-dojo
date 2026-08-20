# Leçon 1 — Exercices

Écrire le code dans `mes-solutions/niveau-01/lecon-01/`, un répertoire par exercice.
**Les corrections sont sur la branche `corrections`** — à n'ouvrir qu'après avoir essayé.

## Exercices faciles

**E1 — Environnement.**
Mettre à jour Go vers 1.27.0, puis relever la sortie de :
```bash
go version
go env GOROOT GOPATH GOMODCACHE
```
Expliquer en une phrase à quoi sert chacune de ces trois variables.
*Chercher soi-même avec `go help environment` — trouver l'information est une compétence à part entière.*

**E2 — Introspection.**
Écrire un programme qui affiche son nom, la version de Go qui l'a compilé, le système
d'exploitation et l'architecture.
*Indice : explorer le paquet `runtime` avec `go doc runtime`, chercher `Version`, `GOOS`, `GOARCH`.*

**E3 — Arguments.**
Afficher les arguments de la ligne de commande, un par ligne, préfixés de leur position :
```
$ go run . pomme poire cerise
1: pomme
2: poire
3: cerise
```
*Indice : une boucle `for … range` est nécessaire.*

**E4 — Lire le compilateur.**
Introduire volontairement trois erreurs, une à la fois, et **relever le message exact** :
(a) importer `"strings"` sans l'utiliser ;
(b) déclarer `x := 5` sans jamais utiliser `x` ;
(c) placer l'accolade ouvrante de `main` sur la ligne suivante.
*Savoir lire un message d'erreur Go est une compétence à part entière ; le compilateur Go
est parmi les plus clairs qui existent.*

**E5 — Le binaire.**
```bash
go build -o bin/args .
ls -lh bin/args
./bin/args a b c
```
Quelle est la taille du binaire ? Pourquoi un programme aussi trivial pèse-t-il autant ?
*Réfléchir à ce que « pas de runtime à installer » implique.*

---

## Exercice intermédiaire — `tempconv`

Convertisseur de température en ligne de commande.

```
$ go run . 100 C
100.0°C = 212.0°F
$ go run . 32 F
32.0°F = 0.0°C
$ go run .
usage: tempconv <valeur> <C|F>
```

**Contraintes :**

1. Sans argument : message d'usage sur **`os.Stderr`** (pas `os.Stdout`) et sortie avec le **code 1** (`os.Exit(1)`). Chercher pourquoi cette distinction compte en shell — c'est une exigence de « prêt pour la production », pas une coquetterie.
2. L'unité est acceptée en majuscule comme en minuscule.
3. Une valeur non numérique produit un message d'erreur clair, pas un plantage.
4. La conversion vit dans **une fonction séparée** de `main`, qui **n'affiche rien**.
5. Exactement une décimale à l'affichage.

*Indices : `strconv.ParseFloat`, `strings.ToUpper`, `fmt.Fprintf` (le `F` = *File* : écrit
vers une destination au choix), le verbe `%.1f`.*

**Le point caché de l'exercice :** séparer le calcul des entrées/sorties. C'est le
fondement du niveau 3 (tests) et du niveau 7 (architecture). Une fonction qui calcule
*et* qui affiche est presque intestable.

---

## Défi

Rendre `tempconv` capable de traiter **plusieurs conversions en une invocation**, en
lisant les arguments, ou **l'entrée standard** si aucun argument n'est fourni :

```bash
$ printf "100 C\n32 F\n-40 C\n" | go run .
100.0°C = 212.0°F
32.0°F = 0.0°C
-40.0°C = -40.0°F
```

**Contraintes supplémentaires :**
- une ligne invalide n'interrompt pas le traitement : message sur `os.Stderr`, on continue ;
- si **au moins une** ligne était invalide, le code de sortie final est 1, sinon 0 ;
- le programme doit traiter un flux de 10 millions de lignes sans faire exploser la mémoire.

*Indice unique : `bufio.Scanner`.* La dernière contrainte est celle qui distingue « ça
fonctionne » de « c'est prêt pour la production ». Quelle est la différence entre lire
tout le fichier en mémoire et le lire ligne par ligne ?

Ce défi est volontairement au-dessus du niveau attendu après une seule leçon.
Échouer dessus est normal — montrer où ça bloque est l'objectif.

---

## Quiz

À faire **sans exécuter de code ni relire le cours**.

1. Vrai ou faux : deux fichiers `.go` du même répertoire peuvent déclarer des packages différents. Justifier.
2. Quelle est la différence entre `go run .` et `go build` ?
3. Pourquoi `func Add()` et `func add()` n'ont-elles pas la même visibilité ?
4. Importer `"strings"` sans l'utiliser : avertissement ou erreur ? Pourquoi ce choix ?
5. À quoi sert la ligne `go 1.27` dans `go.mod` ? *(La réponse évidente est fausse.)*
6. Différence entre `fmt.Printf` et `fmt.Sprintf` ?
7. `go vet` détecte-t-il des erreurs que le compilateur laisse passer ? Donner un exemple.
8. Que contient `os.Args[0]` ?
