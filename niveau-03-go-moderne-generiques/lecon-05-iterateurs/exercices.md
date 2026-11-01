# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-03/lecon-05/`.

## Exercices faciles

**E1 — Le corps devient `yield`.**
Écrire `Range(lo, hi int) iter.Seq[int]` qui produit `lo, lo+1, …, hi-1`. Ajouter un
`fmt.Println("début")` au début de la fonction **interne** et un `fmt.Println("fin")` juste
avant chaque `return`. Consommer avec `for i := range Range(0, 10) { if i == 2 { break } … }`.
Prédire **la sortie exacte** avant d'exécuter, puis comparer avec `cat -A`.

**E2 — Le contrat violé.**
Écrire un itérateur qui ignore le résultat de `yield`. Le consommer **sans** `break` : que se
passe-t-il ? Puis **avec** `break` : recopier le message exact. Pourquoi le runtime ne peut-il
détecter le bug que dans le second cas ?

**E3 — Clés triées.**
Remplacer ce code par une seule ligne :

```go
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
sort.Strings(keys)
```

Puis faire de même pour obtenir les **valeurs** triées par ordre décroissant (indice :
`slices.SortedFunc`).

**E4 — `Seq2`.**
Écrire `Enumerate[V any](seq iter.Seq[V]) iter.Seq2[int, V]` qui numérote les éléments d'un
itérateur à partir de 0. L'appliquer à `strings.FieldsSeq("  a b   c ")`. Sortie attendue :

```
0 a
1 b
2 c
```

**E5 — Usage unique.**
Créer un `WordScanner` (celui du cours) et parcourir **deux fois** `ws.Words()`. Combien de
mots au second parcours ? Pourquoi ? Comment un utilisateur de l'API pourrait-il le savoir
sans lire le code ?

---

## Exercice intermédiaire — `logscan`, analyser un journal sans tout charger

On reçoit un journal au format :

```
2026-10-03T08:12:01Z INFO  api    requête GET /users 12ms
2026-10-03T08:12:02Z ERROR db     connexion refusée
2026-10-03T08:12:02Z WARN  api    requête lente GET /search 950ms
```

Écrire un paquet `logscan` avec :

```go
type Entry struct {
	Time    string   // on ne parse pas encore les dates (niveau 4, leçon 4)
	Level   string
	Service string
	Message string
}

// Entries itère sur les entrées. Une ligne mal formée produit une erreur
// pour CETTE ligne (avec son numéro), puis l'itération continue.
func Entries(r io.Reader) iter.Seq2[Entry, error]

func ByLevel(seq iter.Seq2[Entry, error], level string) iter.Seq2[Entry, error]
```

**Contraintes :**
1. **Aucun** slice de toutes les lignes en mémoire : le programme doit traiter un journal de
   plusieurs gigaoctets.
2. `ByLevel` laisse **passer les erreurs** (on ne veut pas les perdre en filtrant).
3. Une erreur de **lecture** du flux (pas une ligne mal formée) est produite une dernière
   fois, puis l'itération s'arrête.
4. Le `main` lit `os.Stdin`, affiche les `ERROR`, puis un résumé : nombre d'entrées par niveau
   (ordre alphabétique) et nombre de lignes mal formées.
5. Le `main` s'arrête au **5ᵉ** `ERROR` rencontré. Vérifier, avec un compteur placé dans
   `Entries`, que les lignes suivantes ne sont **pas** lues.
6. **Question de conception** : on a choisi `Seq2[Entry, error]`. Réécrire la signature dans
   le style `bufio.Scanner` (`Err()`). Avec ce style, comment remonter les lignes mal formées,
   qui ne doivent **pas** arrêter l'itération ? Conclure en trois lignes sur le choix.

---

## Défi — Fusionner deux itérateurs triés

```go
func Merge[V cmp.Ordered](a, b iter.Seq[V]) iter.Seq[V]
```

Les deux itérateurs produisent des valeurs **croissantes** ; `Merge` produit leur fusion
croissante (comme l'étape de fusion du tri fusion), **sans** les collecter dans des slices.

1. Pourquoi est-ce impossible à écrire avec deux `for … range` imbriqués ou successifs ?
2. Écrire `Merge` avec `iter.Pull`. Où placer les deux `stop()` ? Que se passe-t-il si le
   consommateur de `Merge` fait `break` au milieu ?
3. Tester avec `Merge(slices.Values([]int{1, 4, 9}), slices.Values([]int{2, 3, 10, 11}))`.
4. Tester avec deux itérateurs **infinis** (les multiples de 3 et de 5) et un `Take(…, 10)`.
   Sortie attendue : `[0 0 3 5 6 9 10 12 15 15]`.

---

## Quiz

1. Quelle est la définition de `iter.Seq[V]` ?
2. Que devient le corps d'une boucle `for v := range seq` ?
3. Que doit faire un itérateur quand `yield` rend `false` ? Que se passe-t-il sinon ?
4. Pourquoi `for i, w := range strings.SplitSeq(s, ",")` ne compile-t-il pas ?
5. Comment obtenir les clés triées d'une map en une ligne ?
6. Avantage de `strings.SplitSeq` sur `strings.Split` ?
7. Quelles sont les deux façons de remonter une erreur pendant une itération, et le risque de chacune ?
8. Qu'est-ce qu'un itérateur à usage unique ? Donner un exemple.
9. À quoi sert `iter.Pull`, et quelle obligation impose-t-il ?
10. Quand vaut-il mieux rendre un `[]T` qu'un `iter.Seq[T]` ?
