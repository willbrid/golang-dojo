# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-06/lecon-04/`. Tout avec `-race`.

## Exercices faciles

**E1 — Le hasard de `select`.**
Deux channels bufferisés de capacité 1 000, remplis chacun de 1 000 valeurs. Une boucle de 1 000
`select` lit l'un ou l'autre. Compter combien de fois chaque cas est choisi. Recommencer 5 fois. Le
premier cas écrit est-il favorisé ?

**E2 — Délai.**
Écrire `func FirstOf(timeout time.Duration, srcs ...func() string) (string, error)` qui lance
chaque fonction dans une goroutine et rend le **premier** résultat arrivé, ou une erreur après
`timeout`. Les goroutines perdantes ne doivent pas fuir : vérifier avec `runtime.NumGoroutine()`
après une pause.

**E3 — Le `default` qui brûle le CPU.**
Écrire une boucle `for { select { case v := <-ch: …; default: } }` qui attend une valeur envoyée
après 2 s. Mesurer l'usage CPU (`time go run .` : comparer `user` et `real`). Corriger.

**E4 — Le channel fermé toujours prêt.**
Dans `mergeTwo` du cours, retirer les `a = nil` et `b = nil` (garder le `continue`) et compter le
nombre de tours de boucle. Que se passe-t-il ? Pourquoi ?

**E5 — Arrêter plusieurs goroutines.**
Lancer 5 goroutines-boucles de service qui écoutent un même `quit`. Les arrêter d'abord en
**envoyant** une valeur sur `quit` : combien s'arrêtent ? Puis en **fermant** `quit`. Attendre leur
fin avec un `WaitGroup` et afficher le nombre de goroutines restantes.

---

## Exercice intermédiaire — Un limiteur de débit

Écrire un type `Limiter` qui autorise au plus `n` événements par seconde, **sans** paquet externe :

```go
func NewLimiter(perSecond int) *Limiter
func (l *Limiter) Wait()           // bloque jusqu'à ce qu'un jeton soit disponible
func (l *Limiter) TryAcquire() bool // ne bloque jamais
func (l *Limiter) Stop()
```

**Contraintes :**
1. Implémentation par **jetons** dans un channel bufferisé de capacité `perSecond`, rechargé par une
   goroutine avec un `time.Ticker`. `Wait` reçoit un jeton ; `TryAcquire` utilise `default`.
2. La goroutine de recharge s'arrête proprement avec `Stop` (boucle de service avec `quit`).
3. La recharge ne doit **jamais** bloquer quand le seau est plein (indice : `select` + `default`).
4. Mesurer : 100 appels à `Wait` avec `perSecond = 20` durent environ combien de temps ? Expliquer
   pourquoi les 20 premiers passent immédiatement (rafale autorisée).
5. Écrire un test avec `testing/synctest` (Go 1.25) qui vérifie le débit **sans attendre** en temps
   réel. (Voir le défi pour la mise en route.)

---

## Défi — Tester le temps sans l'attendre : `testing/synctest`

Le paquet `testing/synctest` exécute une fonction dans une **bulle** où l'horloge est **virtuelle** :
`time.Sleep`, `time.After`, les tickers avancent instantanément dès que toutes les goroutines de la
bulle sont bloquées.

1. Lire `go doc testing/synctest`. Écrire un test de `withTimeout` (cours) qui vérifie qu'un appel
   lent échoue **après exactement 2 secondes virtuelles** (`time.Since(start) == 2*time.Second`) et
   qu'un appel rapide réussit. Le test doit durer quelques millisecondes réelles.
2. Lancer le test : il échoue avec `deadlock: main bubble goroutine has exited but blocked goroutines
   remain`. Quelle goroutine est encore bloquée ? Qu'est-ce que ça dit du code testé ?
3. Corriger le **test** (pas le code) pour qu'il passe, en expliquant pourquoi c'est légitime ici.
4. Que faudrait-il changer dans `fetch` pour que le travail abandonné s'arrête vraiment ? (Réponse
   complète à la leçon 8.)

---

## Quiz

1. Que fait `select` quand plusieurs cas sont prêts ?
2. Que fait-il quand aucun n'est prêt et qu'il n'y a pas de `default` ?
3. Comment poser un délai sur une réception ?
4. Un délai expiré arrête-t-il le travail en cours ?
5. À quoi sert `default`, et quand est-il dangereux ?
6. Pourquoi mettre à `nil` un channel fermé dans un `select` en boucle ?
7. Pourquoi arrêter des goroutines en **fermant** un channel plutôt qu'en envoyant ?
8. Que faut-il faire d'un `time.Ticker` quand on n'en a plus besoin ?
9. Que fait `select {}` ?
10. Qu'apporte `testing/synctest` ?
