# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-11/lecon-02/`, en partant du paquet de l'exemple.

## Exercices faciles

**E1 — Lire les raisons.**
Lancer `go build -gcflags='-m -m'` sur l'exemple. Pour les cas 2, 5 et 8, recopier la chaîne de raisons que
donne le compilateur (« flow: … »). Laquelle est la plus difficile à comprendre ?

**E2 — L'*inlining* change tout.**
Retirer `//go:noinline` de `DynamicBuffer`. Que deviennent les allocations de `DynamicBuffer(33)` ? Et
`NewPointer` : ajouter `//go:noinline` au-dessus, puis mesurer. Expliquer pourquoi le même code alloue ou
non selon qu'il est *inliné*.

**E3 — La frontière des 32 octets.**
Mesurer `DynamicBuffer(n)` pour n = 1, 16, 31, 32, 33, 64, 1 024, 65 536. Tracer `allocs/op` et `B/op`.
Pourquoi `B/op` vaut-il 48 pour n = 33 ? (Indice : les classes de taille de l'allocateur, leçon 3.)

**E4 — Le `fmt` caché.**
Écrire trois versions de `func Key(id int, kind string) string` : `fmt.Sprintf("%s:%d", kind, id)`,
`kind + ":" + strconv.Itoa(id)`, et avec `strconv.AppendInt` dans un tableau de 32 octets. Mesurer temps et
allocations. Laquelle utiliser dans une clé de cache appelée un million de fois par seconde ?

**E5 — L'échappement par un canal.**
Envoyer `&Point{}` dans un canal bufferisé, puis le recevoir dans la même fonction. Le compilateur peut-il
prouver que la valeur ne s'échappe pas ? Mesurer.

---

## Exercice intermédiaire — Un encodeur sans allocation

Une fonction de journalisation encode des événements `Event{Time time.Time; Level int; Msg string;
Attrs map[string]string}` en une ligne `2026-03-01T10:00:00Z niveau=2 msg="…" clé=valeur…`.

1. Écrire la version naïve avec `fmt.Sprintf` et `strings.Join` ; mesurer.
2. Écrire `func AppendEvent(dst []byte, e Event) []byte` qui ne fait **aucune** allocation quand `dst` a
   assez de capacité (`time.Time.AppendFormat`, `strconv.AppendInt`, `strconv.AppendQuote`).
3. Les clés de `Attrs` doivent sortir **triées** : comment le faire sans allouer à chaque appel ? (Une slice
   de clés réutilisée par l'appelant, ou des attributs fournis déjà triés : choisir.)
4. Vérifier : `allocs/op = 0` quand le tampon est réutilisé ; même sortie que la version naïve (table de
   tests et fuzzing).
5. Lire `-gcflags=-m` pour ta version : quelles valeurs restaient sur le tas avant ta dernière
   modification ?

---

## Défi — La pile qui grandit

1. Écrire une fonction récursive de profondeur 1 000 000 dont chaque appel a une variable locale de 1 Kio.
   Mesurer le temps et la mémoire (`runtime.ReadMemStats`, `StackInuse`).
2. La même avec 100 000 goroutines qui ont chacune une profondeur de 10. Comparer `StackInuse`.
3. Lire la documentation de `runtime/debug.SetMaxStack` : que se passe-t-il au-delà ? Provoquer l'erreur
   (`goroutine stack exceeds … limit`) et noter qu'on ne peut pas la rattraper par `recover`.
4. Expliquer pourquoi une pile qui **se recopie** pour grandir empêche de garder un pointeur vers la pile
   d'une autre goroutine — et pourquoi c'est l'une des raisons de l'analyse d'échappement.

---

## Quiz

1. Pourquoi une allocation sur la pile coûte-t-elle presque rien ?
2. Quel est le vrai coût d'une allocation sur le tas ?
3. Qui décide où vit une valeur, et selon quelle règle ?
4. Que signifient `escapes to heap`, `moved to heap`, `leaking param` ?
5. Pourquoi convertir une valeur en `any` la fait-elle souvent s'échapper ?
6. Pourquoi une variable capturée par une closure rendue part-elle sur le tas ?
7. `does not escape` garantit-il l'absence d'allocation ?
8. Quel rôle joue l'*inlining* dans l'analyse d'échappement ?
9. Comment écrire une fonction qui produit des octets sans allouer ?
10. Quand faut-il se préoccuper de l'échappement ?
