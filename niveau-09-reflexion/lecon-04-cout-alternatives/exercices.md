# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-09/lecon-04/`, en partant du module de l'exemple.

## Exercices faciles

**E1 — Reproduire les mesures.**
Écrire les quatre versions de la somme du champ `Price` (directe dans une fonction, générique,
réflexion par nom, réflexion par index) et les mesurer avec `-benchmem -count 6`. Écrire ensuite la
boucle directe dans le corps du benchmark, sur une variable globale : retrouves-tu l'écart du cours ?
Que montre `go build -gcflags=-m` pour les deux versions ? (Le niveau 11 approfondit ; ici, constater.)

**E2 — Oublier de régénérer.**
Ajouter un champ `ISBN string \`db:"isbn"\`` à `Book` **sans** relancer `go generate`. Le code
compile-t-il ? Quel test échoue, avec quel message ? Écrire la commande qui, en CI, échoue si un fichier
généré n'est pas à jour.

**E3 — Le générateur se trompe.**
Introduire une erreur dans le modèle de génération (une virgule de trop, une accolade manquante). Que se
passe-t-il avec `format.Source` ? Et sans lui ? Pourquoi est-il préférable que le **générateur** échoue
plutôt que la compilation du paquet généré ?

**E4 — `stringer`.**
Écrire à la main ce que génère l'outil officiel `stringer` pour un type
`type Status int` avec trois constantes (`Draft`, `Published`, `Archived`) : une méthode
`String() string`. Puis l'écrire avec un générateur minimal (`go/parser` sur les constantes d'un type).
Pourquoi `stringer` génère-t-il aussi une fonction qui **ne compile plus** si les valeurs des constantes
changent ? (Lire un fichier `_string.go` généré par l'outil si tu l'as installé :
`go install golang.org/x/tools/cmd/stringer@latest`.)

**E5 — Le bon outil.**
Pour chacun de ces besoins, choisir entre interface, génériques, génération et réflexion, en une
phrase : (a) un cache LRU pour n'importe quel type de valeur ; (b) l'encodage JSON de réponses HTTP de
types variés ; (c) les méthodes `Scan` de 40 tables d'un service à fort trafic ; (d) un validateur
`validate:"min=1,max=200"` pour les entrées d'une API ; (e) un journal qui accepte « n'importe quoi »
comme valeur ; (f) des *mocks* pour 15 interfaces.

---

## Exercice intermédiaire — Un générateur complet pour le niveau 8

Étendre `dbgen` pour produire, pour chaque struct marquée par un commentaire `//dbgen:table books` :

1. `ColumnsBook`, `ScanDest()` (déjà faits) ;
2. `func (x *Book) InsertArgs() []any` (sans la colonne `id`, marquée `db:"id,auto"`) et la constante
   `InsertBook` (`INSERT INTO books (…) VALUES ($1, …) RETURNING id`) ;
3. `func ScanBooks(rows *sql.Rows) ([]Book, error)` qui applique le patron complet de N8 L2 (`Close`,
   `Err`) ;
4. une erreur de génération claire pour : balise `db` en double, struct marquée sans aucun champ `db`,
   type de champ non géré (`chan`, `func`) ;
5. des tests du générateur lui-même : un fichier source d'entrée en mémoire (`parser.ParseFile` accepte une
   chaîne), et le code attendu (fichier « golden », N5 L4).

---

## Défi — Réflexion contre génération, sur un vrai encodeur

1. Écrire un encodeur JSON **par réflexion**, minimal (structs à champs `string`, `int`, `float64`,
   `bool`, slices de ces types ; balise `json`), avec cache de l'analyse par type.
2. Écrire le générateur qui produit, pour un type donné, une méthode `AppendJSON(b []byte) []byte`
   équivalente, sans réflexion.
3. Vérifier que les deux produisent exactement la même sortie que `encoding/json` sur une table de valeurs,
   puis par fuzzing (chaînes avec guillemets, caractères de contrôle, Unicode).
4. Mesurer les trois (réflexion maison, généré, `encoding/json`) sur 10 000 objets : temps et allocations.
5. Conclure : dans quels cas un service devrait-il générer son code JSON plutôt qu'utiliser
   `encoding/json` ? Quels coûts de maintenance cela ajoute-t-il ?

---

## Quiz

1. D'où vient l'essentiel du coût de la réflexion, et comment le réduire ?
2. Que savent faire les génériques que la réflexion faisait avant Go 1.18 ? Que ne savent-ils pas faire ?
3. Qu'est-ce que `go generate`, et quand s'exécute-t-il ?
4. Pourquoi versionner le code généré ?
5. Quelle est la forme exacte de l'en-tête d'un fichier généré, et à quoi sert-elle ?
6. À quoi sert `go/format.Source` dans un générateur ?
7. Comment s'assurer en CI que le code généré est à jour ?
8. Citer trois générateurs de code de l'écosystème Go.
9. Dans quels cas la réflexion reste-t-elle le bon choix ?
10. Pourquoi mesurer le code dans une fonction plutôt que dans le corps du benchmark ?
