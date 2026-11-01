# Leçon 8 — Exercices

Code dans `mes-solutions/niveau-06/lecon-08/`. Tout avec `-race` ; les tests de temps avec
`testing/synctest`.

## Exercices faciles

**E1 — `lostcancel`.**
Écrire `ctx, _ := context.WithTimeout(context.Background(), time.Second)`. Recopier le message de
`go vet`. Puis écrire une fonction qui rend le contexte **sans** appeler `cancel` ni le rendre :
`vet` le voit-il encore ? Que faut-il faire si une fonction crée un contexte qui doit vivre plus
longtemps qu'elle ?

**E2 — Réparer `fetch`.**
Reprendre `fetch` et `withTimeout` de la leçon 4. Les réécrire pour que `fetch` reçoive un `ctx` et
s'arrête quand il est annulé. Reprendre le test `synctest` de la leçon 4 : le `time.Sleep(time.Hour)`
final est-il encore nécessaire ? Pourquoi ?

**E3 — Une boucle de calcul annulable.**
`countPrimes(lo, hi)` (leçon 1) ne bloque jamais : un `select` n'a pas de sens. La rendre annulable
avec un test de `ctx.Err()` tous les 10 000 nombres. Mesurer : combien de temps entre l'annulation et
le retour effectif ? Que se passe-t-il si on teste `ctx.Err()` à **chaque** nombre ?

**E4 — La chaîne coupée.**
Trouver le bug :

```go
func Handle(ctx context.Context, id string) error {
	user, err := loadUser(context.Background(), id)
	…
}
```

Écrire un test qui le révèle (un `ctx` déjà annulé doit faire échouer `Handle` immédiatement).

**E5 — Clé de valeur.**
Deux paquets `auth` et `trace` utilisent chacun `context.WithValue(ctx, "id", …)`. Montrer la
collision. Corriger avec des clés typées et des accesseurs. `go vet` signale-t-il la clé de type
`string` ? *(Essayer aussi `staticcheck`, règle `SA1029`.)*

---

## Exercice intermédiaire — Recherche avec budget

Un service interroge trois sources (simulées par des fonctions qui dorment une durée aléatoire et
respectent leur `ctx`) et doit répondre en **200 ms au plus**.

```go
func Search(ctx context.Context, query string) ([]Result, error)
```

**Contraintes :**
1. Les trois sources sont interrogées **en parallèle**, chacune avec le contexte reçu.
2. Si une source échoue, les **autres sont annulées** immédiatement (`WithCancelCause`, la cause
   étant l'erreur de la source).
3. Si le budget de 200 ms (dérivé du `ctx` reçu, qui peut avoir un délai plus court) expire, rendre
   une erreur qui permet `errors.Is(err, context.DeadlineExceeded)`.
4. **Aucune goroutine ne survit** au retour de `Search`, dans aucun cas : le vérifier dans les tests
   avec `synctest` (qui échoue si une goroutine reste bloquée).
5. Table de tests (synctest) : toutes rapides ; une lente au-delà du budget ; une en erreur ; un `ctx`
   parent déjà annulé ; un `ctx` parent avec un délai de 50 ms.
6. Garder le code : la leçon 9 le réécrira avec `errgroup` en quelques lignes.

---

## Défi — Implémenter `WithTimeout`

Sans utiliser `context.WithTimeout`/`WithDeadline`/`WithCancel`, écrire une fonction :

```go
func MyWithTimeout(parent context.Context, d time.Duration) (context.Context, context.CancelFunc)
```

1. Écrire un type qui implémente l'interface `context.Context` (quatre méthodes).
2. `Done()` est fermé quand `d` expire, quand `cancel` est appelé, **ou** quand le parent est annulé.
   Indice : `context.AfterFunc(parent, f)` (Go 1.21) appelle `f` à l'annulation du parent sans goroutine
   permanente.
3. `cancel` peut être appelé plusieurs fois, et depuis plusieurs goroutines, sans panique (`sync.Once`).
4. `Err()` rend la bonne erreur selon la cause. `Value` délègue au parent.
5. Tester avec `synctest`, et comparer avec le comportement de `context.WithTimeout` sur les mêmes
   cas. Lire ensuite `go doc -src context.WithDeadline` : qu'est-ce que la vraie implémentation fait
   de plus ?

---

## Quiz

1. Que représente `ctx.Done()` ?
2. Quelles valeurs peut rendre `ctx.Err()` ?
3. Pourquoi faut-il toujours appeler la fonction `cancel` ?
4. Un enfant peut-il avoir un délai plus long que son parent ?
5. Que doit faire une fonction qui reçoit un `ctx` ?
6. Où placer le paramètre `ctx`, et pourquoi ne pas le stocker dans une struct ?
7. Quand utiliser `context.TODO()` ?
8. À quoi sert `context.Cause` ?
9. Que peut-on mettre dans les valeurs d'un contexte, et que ne doit-on pas y mettre ?
10. Pourquoi une clé de valeur doit-elle être d'un type non exporté ?
