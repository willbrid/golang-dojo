# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-09/lecon-03/`.

## Exercices faciles

**E1 — Valeur ou pointeur.**
Appeler `Call(*lib, "Add", "X", "1")` (une **copie** de la bibliothèque) au lieu de `Call(lib, …)`. Que
rend `MethodByName` ? Pourquoi ? Et si `Add` avait un récepteur valeur, l'ajout serait-il visible dans
`lib` ?

**E2 — Le mauvais type d'argument.**
Construire les arguments avec `reflect.ValueOf(raw)` (une `string`) sans conversion, et appeler `Count`
puis `Add`. Relever le message de panique de `Call`. Puis passer `reflect.ValueOf(3)` à une méthode qui
attend un `int64` : le message ?

**E3 — Signature par le type.**
Comparer `reflect.TypeFor[*Library]().MethodByName("Add")` et
`reflect.ValueOf(lib).MethodByName("Add").Type()` : que vaut `NumIn` dans chaque cas ? Appeler la méthode
obtenue par le **type** (`Method.Func.Call`) : que faut-il passer en premier argument ?

**E4 — `DeepEqual` dans tes tests.**
Écrire un test qui compare, avec `reflect.DeepEqual`, une réponse JSON décodée contenant une liste vide
(`"tags": []`) à une valeur attendue écrite `Tags: nil`. Le voir échouer, puis le corriger de trois
façons : attendu `Tags: []string{}`, `slices.Equal` champ par champ, et une fonction de comparaison qui
traite nil et vide comme égaux. Laquelle préférer ?

**E5 — Mesurer `Traced`.**
Benchmark (N5 L6) : `add(2, 3)` direct contre `Traced("add", add)(2, 3)` (sans le `Printf` de trace).
Combien de fois plus lent ? Combien d'allocations par appel ? Qu'est-ce qui alloue ?

---

## Exercice intermédiaire — Un répartiteur sûr

Transformer `Call` en un répartiteur de commandes pour une CLI ou une API d'administration :

1. `NewDispatcher(target any, allowed ...string) (*Dispatcher, error)` vérifie **à la construction** que
   chaque méthode autorisée existe, que ses paramètres sont de sortes gérées (`string`, entiers, `bool`,
   `time.Duration`), et qu'elle rend au plus une valeur plus une `error`. Toute anomalie est une erreur
   de construction, pas d'appel.
2. `Dispatch(line string)` découpe la ligne (`strings.Fields`), appelle la méthode, et rend le résultat
   formaté.
3. Une méthode qui **panique** ne fait pas tomber le répartiteur : la panique est convertie en erreur
   (N6 L12), avec le nom de la méthode.
4. Les méthodes non listées dans `allowed` sont refusées, même si elles sont exportées. Pourquoi est-ce
   indispensable si les lignes viennent du réseau ?
5. Tests : table de lignes valides et invalides, panique, méthode non autorisée.

---

## Défi — Un *mock* universel avec `MakeFunc`

Écrire `func Spy[F any](impl F) (F, *Calls)` qui rend une fonction du même type que `impl`, qui délègue à
`impl` et **enregistre** chaque appel (arguments et résultats) dans `*Calls` (sûr en concurrence).

1. `calls.Len()`, `calls.Args(i) []any`, `calls.Results(i) []any`.
2. Une variante `Stub[F any](results ...any) (F, *Calls)` qui, sans implémentation, rend toujours les
   résultats donnés ; vérifier **à la création** (pas à l'appel) que leurs types correspondent à ceux de
   `F` (y compris une `error` nil).
3. L'utiliser pour tester un service qui reçoit une dépendance sous forme de fonction
   (`type SendMail func(ctx context.Context, to, body string) error`).
4. Comparer avec un faux écrit à la main (N5 L3) : lisibilité, sécurité de type, messages d'erreur. Quand
   le *spy* réflexif vaut-il la peine ?

---

## Quiz

1. Que rend `MethodByName` sur une valeur, et que vaut `NumIn` par rapport à la déclaration de la méthode ?
2. Pourquoi `reflect.ValueOf(lib)` ne voit-il pas les méthodes à récepteur pointeur ?
3. Que se passe-t-il si l'on appelle `Call` avec un argument du mauvais type ?
4. Comment reconnaître qu'une fonction suit la convention « erreur en dernier » ?
5. Que fait `reflect.MakeFunc`, et pourquoi le combiner avec les génériques ?
6. Que rend `reflect.New(t)` ?
7. Pourquoi un registre de constructeurs vaut-il souvent mieux qu'un registre de types ?
8. `reflect.DeepEqual([]int(nil), []int{})` ? Et deux `errors.New("x")` ?
9. Pourquoi ne pas comparer des `time.Time` avec `DeepEqual` ?
10. Que proposer à la place de `DeepEqual` dans les tests ?
