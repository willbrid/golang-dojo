# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-09/lecon-01/`.

## Exercices faciles

**E1 — Type ou sorte ?**
Pour chacune de ces valeurs, prédire `TypeOf(v)` et `TypeOf(v).Kind()`, puis vérifier :
`int8(3)`, `time.Second`, `[3]int{}`, `[]byte("go")`, `errors.New("x")`, `struct{}{}`, `&struct{}{}`,
`make(chan int)`, `fmt.Println`. Pour `errors.New("x")`, pourquoi la sorte n'est-elle pas `Interface` ?

**E2 — Les paniques.**
Écrire cinq lignes qui paniquent chacune pour une raison différente : mauvaise sorte, `Value` invalide,
valeur non modifiable, champ non exporté modifié, `Elem()` sur une sorte qui n'est ni pointeur ni
interface. Les exécuter sous `recover` et relever chaque message. Lequel aurait pu être évité par un test
`Is…` / `Can…` ?

**E3 — Compléter `Describe`.**
Ajouter les sortes `Bool`, `Array`, `Interface` (une struct avec un champ `any`), les entiers non signés,
et les maps triées par clé (l'ordre de `MapRange` est aléatoire : comment s'en apercevoir, et comment
rendre la sortie déterministe ?).

**E4 — Mettre à zéro.**
Écrire `func Zero(ptr any) error` qui remet à la valeur zéro ce que pointe `ptr`, quel qu'en soit le
type (`reflect.Zero`). Rendre une erreur claire si `ptr` n'est pas un pointeur non nil.

**E5 — Le cycle.**
`type Node struct{ Next *Node }`, puis `n := &Node{}; n.Next = n`. Que fait `Describe(n)` ? Corriger en
mémorisant les pointeurs déjà visités (`v.Pointer()` ou `v.UnsafePointer()` comme clé de map).

---

## Exercice intermédiaire — Un `Copy` profond

Écrire `func DeepCopy[T any](v T) T` (signature générique, implémentation réflexive) qui copie
récursivement : slices, maps, pointeurs, structs (champs exportés **et** non exportés : pourquoi est-ce
possible ici alors qu'on ne peut pas les **modifier** ? indice : `reflect.New` puis remplir une valeur
neuve… ou pas ?), en gérant les cycles (E5).

1. Table de tests : la copie est égale à l'original (`reflect.DeepEqual`), et modifier la copie ne
   modifie jamais l'original (vérifier pour chaque sorte).
2. Que faire des champs non exportés ? Essayer, constater la limite, choisir une politique (copie
   superficielle, erreur, panique documentée) et l'écrire dans la documentation.
3. Que faire des `chan`, `func`, `sync.Mutex` ? Pourquoi copier un `sync.Mutex` est-il une faute (N6 L6,
   `go vet`) ?

---

## Défi — Un `fmt.Printf("%v")` minimal

Écrire `func Format(v any) string` qui reproduit le format `%v` de `fmt` pour : nombres, chaînes,
booléens, slices, tableaux, maps (clés triées, comme `fmt`), structs (`{a b}`), pointeurs vers struct
(`&{a b}`), `nil` (`<nil>`), et qui utilise la méthode `String() string` si le type l'implémente
(`fmt.Stringer`, détecté par réflexion **ou** par assertion de type : laquelle choisir, et pourquoi ?).
Tester contre `fmt.Sprintf("%v", v)` sur une table de 30 valeurs, puis par fuzzing sur des valeurs
construites à partir des octets du fuzzer.

---

## Quiz

1. Que contient une valeur d'interface, et que donnent `TypeOf` et `ValueOf` ?
2. Quelle différence entre type et `Kind` ?
3. Pourquoi le code réflexif se branche-t-il sur le `Kind` ?
4. Que se passe-t-il si l'on appelle `v.Int()` sur une `Value` de sorte `String` ?
5. Qu'est-ce qu'une `Value` invalide, et comment l'obtient-on ?
6. Énoncer les trois lois de la réflexion.
7. Pourquoi `reflect.ValueOf(x).SetInt(2)` panique-t-il ?
8. Pourquoi `json.Unmarshal` exige-t-il un pointeur ?
9. Peut-on modifier un champ non exporté par réflexion ?
10. Citer trois usages légitimes de la réflexion, et une alternative à préférer dans le code
    d'application.
