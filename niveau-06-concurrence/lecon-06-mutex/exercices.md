# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-06/lecon-06/`. Tous les tests avec `-race`.

## Exercices faciles

**E1 — Le compteur, trois fois.**
Un compteur incrémenté 100 000 fois par 100 goroutines : (a) sans protection ; (b) avec un
`Mutex` ; (c) avec une goroutine propriétaire et un channel (leçon 2, E5). Vérifier le total,
passer `-race`, et mesurer les trois temps. Classer.

**E2 — La réentrance.**
Écrire un type `Account` avec `Deposit` et `Transfer(to *Account, amount)` où `Transfer` appelle
`Deposit` sur `a` lui-même… sous verrou. Constater l'interblocage, lire la trace. Corriger avec une
méthode interne `depositLocked`.

**E3 — La fuite de référence.**
Ajouter à `Counter` une méthode `All() map[string]int` qui rend `c.counts` directement. Écrire un
test où une goroutine itère sur le résultat pendant que d'autres appellent `Inc`. Que dit `-race` ?
Que peut-il arriver **sans** `-race` ? *(Indice : le runtime détecte parfois l'écriture concurrente
d'une map et arrête le programme.)*

**E4 — Vérifier puis agir.**
Écrire une fonction `Reserve(seats *Seats, n int) bool` qui vérifie qu'il reste `n` places
(`seats.Available()`) puis les réserve (`seats.Take(n)`), chaque méthode prenant le verrou. Lancer
100 goroutines qui réservent 1 place sur 50 disponibles. Combien de réservations réussissent ? Le
détecteur de races dit-il quelque chose ? Corriger.

**E5 — Mesurer `RWMutex`.**
Reproduire les benchmarks du cours, puis ajouter un cas **90 % lectures / 10 % écritures** et un cas
**50 / 50**. À partir de quelle proportion d'écritures `RWMutex` cesse-t-il d'être intéressant ?

---

## Exercice intermédiaire — Un cache avec expiration

```go
type TTLCache[K comparable, V any] struct { … }

func NewTTLCache[K comparable, V any](ttl time.Duration, now func() time.Time) *TTLCache[K, V]
func (c *TTLCache[K, V]) Get(k K) (V, bool)
func (c *TTLCache[K, V]) Set(k K, v V)
func (c *TTLCache[K, V]) Len() int
func (c *TTLCache[K, V]) Purge() int   // supprime les entrées expirées, rend leur nombre
```

**Contraintes :**
1. Générique (N3), horloge injectée (N5 L3).
2. Une entrée expirée n'est **jamais** rendue par `Get`, même si `Purge` n'est pas encore passé.
3. `Purge` ne doit pas bloquer les lecteurs pendant tout son parcours si le cache contient un
   million d'entrées. Proposer une stratégie et la **mesurer** (indices : purger par lots en
   relâchant le verrou entre deux lots ; ou construire la liste des clés à supprimer sous `RLock`,
   puis supprimer sous `Lock` en revérifiant chaque clé).
4. Tests : table pour la logique d'expiration (horloge fausse), test concurrent sous `-race` avec
   lecteurs, écrivains et purges simultanés.
5. Ajouter une goroutine de purge périodique optionnelle, arrêtée proprement par `Close()`
   (leçon 4). La tester avec `testing/synctest`.

---

## Défi — Les philosophes

Cinq philosophes autour d'une table, une fourchette entre chaque paire. Pour manger, un philosophe
prend la fourchette de gauche puis celle de droite (deux `Mutex`).

1. L'implémenter naïvement, chaque philosophe mangeant 1 000 fois. Le programme se bloque-t-il ?
   Combien de temps avant ? Pourquoi ?
2. Corriger en imposant un **ordre global** de prise des verrous (toujours la fourchette de plus
   petit numéro d'abord). Pourquoi cela supprime-t-il l'interblocage ?
3. Une autre solution : un « serveur » qui n'autorise que 4 philosophes à table en même temps
   (sémaphore, leçon 5). Pourquoi fonctionne-t-elle ?
4. Mesurer l'équité : chaque philosophe mange-t-il autant que les autres ? Le `Mutex` de Go
   garantit-il l'équité ? *(Lire `go doc sync.Mutex` : chercher « starvation ».)*

---

## Quiz

1. Que garantit `Unlock` vis-à-vis de la goroutine qui prend ensuite le verrou ?
2. Pourquoi `defer mu.Unlock()` ?
3. Où placer un `Mutex` et les données qu'il protège ?
4. Le `Mutex` de Go est-il réentrant ? Quelle conséquence ?
5. Que se passe-t-il si on déverrouille un verrou libre ?
6. Pourquoi ne jamais rendre la map interne d'un type protégé ?
7. Qu'est-ce que le problème « vérifier puis agir » ? `-race` le détecte-t-il ?
8. Quand `RWMutex` est-il plus rapide que `Mutex` ?
9. Pour quels cas `sync.Map` est-elle conçue ?
10. Que ne faut-il jamais faire sous un verrou ?
