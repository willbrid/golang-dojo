# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-01/lecon-07/`.

## Exercices faciles

**E1 — Map nil.**
Écrire un programme qui, sur une `var m map[string]int` non initialisée, effectue dans
l'ordre : une lecture, un `len`, un `range`, un `delete`, puis une écriture.
**Prédire lesquelles paniquent avant d'exécuter.** Relever le message exact de la panique.

**E2 — Absent ou zéro.**
Construire `map[string]int{"a": 0}` puis écrire une fonction qui distingue correctement
« la clé vaut 0 » de « la clé n'existe pas ». Le démontrer sur `"a"` et `"b"`.

**E3 — Ordre aléatoire.**
Remplir une map de dix clés, la parcourir et afficher les clés. Lancer le programme cinq
fois. Puis écrire la version déterministe. Comparer les deux sorties sur cinq exécutions.

**E4 — Ensemble.**
Écrire `Unique(words []string) []string` qui retourne les mots distincts, **dans leur ordre
de première apparition**. Utiliser une map comme ensemble.
*Attention : « ordre de première apparition » interdit d'itérer sur la map pour construire le résultat.*

**E5 — Inversion.**
Écrire `Invert(m map[string]int) map[int]string`. Que se passe-t-il si deux clés partagent
la même valeur ? Proposer deux comportements possibles, en implémenter un, et **documenter
le choix dans le commentaire de la fonction**.

---

## Exercice intermédiaire — `index`

Construire un index inversé : pour chaque mot, la liste des lignes où il apparaît.

```
$ go run . poeme.txt
amour    : 2, 7, 14
la       : 1, 2, 2, 9      ← ou "1, 2, 9" ? À décider et justifier.
mer      : 3, 11
```

**Contraintes :**
1. Structure de données : `map[string][]int`. Réfléchir à pourquoi cette forme, et pas `map[string]map[int]bool`.
2. Les mots sont normalisés (minuscules, ponctuation retirée).
3. Les mots de moins de 3 caractères sont ignorés.
4. Sortie triée alphabétiquement ; les numéros de ligne croissants.
5. Un mot apparaissant deux fois sur la même ligne : décider si la ligne est listée une ou deux fois, **documenter et implémenter le choix**.
6. Ajouter une commande de recherche : `go run . poeme.txt amour` affiche les lignes complètes contenant le mot, avec leur numéro.
7. Le fichier peut faire 500 Mo : ne jamais le charger entièrement en mémoire.
8. Aucune fonction ne fait à la fois indexation et affichage.

*Indices : `bufio.Scanner`, `strings.Fields`, `slices.Sorted`, `maps.Keys`. Pour la
contrainte 7, se demander ce que l'index lui-même occupe en mémoire — c'est la vraie question.*

---

## Défi

**a) `GroupBy`** — écrire :
```go
func GroupBy(words []string, key func(string) string) map[string][]string
```
qui regroupe des mots par clé calculée. L'utiliser pour grouper par première lettre, puis
par longueur (attention : la clé doit rester une `string`), puis par **anagramme**
(deux mots sont anagrammes s'ils ont les mêmes lettres). Le troisième cas est le vrai
exercice : quelle fonction de clé caractérise un anagramme ?

**b) LRU cache** — implémenter un cache à éviction *least recently used* :
```go
type LRU struct { /* … */ }
func NewLRU(capacity int) (*LRU, error)
func (c *LRU) Get(key string) (int, bool)
func (c *LRU) Put(key string, value int)
func (c *LRU) Len() int
```
**Contraintes :**
- `Get` et `Put` doivent être en **O(1) amorti** — une map seule ne suffit pas, il faut une
  seconde structure pour l'ordre d'usage. Laquelle ? (`container/list` existe, mais essayer
  de raisonner d'abord sur ce qui est nécessaire.)
- Un `Get` compte comme un usage et rafraîchit l'entrée.
- `Put` d'une clé existante met à jour la valeur **et** rafraîchit.
- Capacité 0 ou négative : erreur.
- Écrire dans `main` un scénario prouvant l'éviction correcte sur au moins huit opérations.

*Le LRU est l'exercice d'entretien le plus demandé au monde. Le faire une fois
sérieusement, sans regarder de solution, vaut dix exercices faciles.*

---

## Quiz

1. Que retourne `m["absent"]` sur une `map[string]int` ?
2. Quelle opération panique sur une map `nil` ? Lesquelles sont sûres ?
3. À quoi sert le second résultat de `v, ok := m[k]` ?
4. L'ordre d'itération d'une map est-il aléatoire par accident ou par conception ?
5. Quels types ne peuvent **pas** servir de clé ?
6. `&m["k"]` compile-t-il ? Pourquoi ?
7. Que se passe-t-il si deux goroutines écrivent en même temps dans une map ?
8. `delete(m, k)` sur une clé absente : erreur, panique, ou rien ?
9. Différence entre `map[string]bool` et `map[string]struct{}` pour représenter un ensemble ?
10. Passer une map à une fonction qui la modifie : l'appelant voit-il les changements ?
