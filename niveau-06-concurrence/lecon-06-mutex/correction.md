# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le compteur, trois fois

| Version | Total | `-race` | Ordre de grandeur |
|---|---|---|---|
| sans protection | faux, variable | race signalée | le plus rapide… et faux |
| `Mutex` | 100 000 | silencieux | ~20–80 ns par incrément selon la contention |
| goroutine + channel | 100 000 | silencieux | ~100–300 ns par incrément (un rendez-vous chacun) |

Pour un simple compteur, le `Mutex` est **plus simple et plus rapide** que le channel ; `atomic`
(leçon 7) le sera encore plus. La goroutine propriétaire se justifie quand l'état et ses règles sont
complexes, pas pour un entier.

## E2 — La réentrance

```go
func (a *Account) Deposit(n int) {
	a.mu.Lock()
	defer a.mu.Unlock()
	a.depositLocked(n)
}

// depositLocked suppose a.mu déjà pris.
func (a *Account) depositLocked(n int) { a.balance += n }

func (a *Account) Refund(n int) {      // une méthode qui réutilise Deposit sous verrou
	a.mu.Lock()
	defer a.mu.Unlock()
	a.depositLocked(n)                  // et non a.Deposit(n) : interblocage
}
```

Avec `a.Deposit(n)` sous verrou, la trace montre la goroutine bloquée sur `[sync.Mutex.Lock]` à
l'intérieur de `Deposit`, appelée depuis la méthode qui tient déjà le verrou. Le suffixe `Locked` est
une convention de la bibliothèque standard : il dit au lecteur « appelez-moi seulement verrou pris ».

Pour `Transfer(to, n)` entre **deux** comptes, il faut prendre **deux** verrous : toujours dans le
même ordre (par exemple par identifiant croissant), sinon deux virements croisés A→B et B→A
s'interbloquent — c'est le défi des philosophes.

## E3 — La fuite de référence

`-race` signale une lecture (l'itération) concurrente d'écritures (`Inc`). Sans `-race`, le runtime
détecte souvent l'accès concurrent à la map et arrête le programme :

```
fatal error: concurrent map iteration and map write
```

C'est une **erreur fatale**, pas une panique : `recover` ne peut rien. La seule correction est de
rendre une copie, construite sous verrou.

## E4 — Vérifier puis agir

Résultat typique : **plus de 50** réservations réussissent (52, 57…), et `-race` ne dit **rien** :
chaque accès est protégé, il n'y a aucune data race. Le bug est **logique** : entre `Available()` et
`Take()`, d'autres goroutines prennent des places. Correction : une seule méthode atomique.

```go
func (s *Seats) TryTake(n int) bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.available < n {
		return false
	}
	s.available -= n
	return true
}
```

Exactement 50 succès. Règle : une décision et l'action qui en dépend vivent **sous le même verrou**.

## E5 — Mesurer `RWMutex`

Le gain de `RWMutex` fond très vite avec les écritures : chaque `Lock` attend que **tous** les
lecteurs sortent et bloque tous les nouveaux. Ordre de grandeur courant : net avantage à 100 % de
lectures, gain faible ou nul vers 90/10, `Mutex` égal ou meilleur à 50/50. Les chiffres dépendent de
la machine et de la durée des sections critiques — le but est d'avoir **mesuré** la tienne avant de
choisir.

## Exercice intermédiaire — Un cache avec expiration

```go
type entry[V any] struct {
	v       V
	expires time.Time
}

type TTLCache[K comparable, V any] struct {
	ttl  time.Duration
	now  func() time.Time
	mu   sync.RWMutex
	data map[K]entry[V] // protégé par mu
}

func NewTTLCache[K comparable, V any](ttl time.Duration, now func() time.Time) *TTLCache[K, V] {
	return &TTLCache[K, V]{ttl: ttl, now: now, data: make(map[K]entry[V])}
}

func (c *TTLCache[K, V]) Get(k K) (V, bool) {
	c.mu.RLock()
	e, ok := c.data[k]
	c.mu.RUnlock()
	if !ok || !c.now().Before(e.expires) {   // expirée = absente, même avant Purge
		var zero V
		return zero, false
	}
	return e.v, true
}

func (c *TTLCache[K, V]) Set(k K, v V) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.data[k] = entry[V]{v: v, expires: c.now().Add(c.ttl)}
}

// Purge en deux temps : repérer sous RLock (les lecteurs continuent),
// supprimer sous Lock en revérifiant (une clé a pu être rafraîchie entre-temps).
func (c *TTLCache[K, V]) Purge() int {
	now := c.now()
	var expired []K
	c.mu.RLock()
	for k, e := range c.data {
		if !now.Before(e.expires) {
			expired = append(expired, k)
		}
	}
	c.mu.RUnlock()

	c.mu.Lock()
	defer c.mu.Unlock()
	n := 0
	for _, k := range expired {
		if e, ok := c.data[k]; ok && !now.Before(e.expires) {   // revérification
			delete(c.data, k)
			n++
		}
	}
	return n
}
```

- **Contrainte 2** : `Get` compare l'expiration lui-même ; `Purge` n'est qu'une question de mémoire.
- **Contrainte 3** : le parcours (long) se fait sous `RLock`, qui n'empêche pas les autres lecteurs ;
  seule la suppression prend le verrou exclusif. La **revérification** évite de supprimer une clé
  qu'un `Set` aurait rafraîchie entre les deux phases — c'est encore « vérifier puis agir ».
- `c.now()` est appelé **hors** verrou : l'horloge injectée pourrait être lente ou elle-même verrouillée.
- La purge périodique : une boucle de service avec `time.NewTicker` et un channel `quit` fermé par
  `Close`, attendue par un `done` (leçon 4). Avec `synctest`, `time.Sleep(10 * ttl)` dans la bulle
  fait passer dix purges en quelques microsecondes.

## Défi — Les philosophes

1. La version naïve se bloque tôt ou tard — parfois au bout de quelques itérations, parfois de
   milliers : quand les cinq prennent leur fourchette gauche **au même moment**, chacun attend la
   droite, tenue par son voisin. Cycle d'attente = interblocage. Le runtime le détecte
   (`all goroutines are asleep`) puisque toutes les goroutines sont bloquées.
2. **Ordre global** : avec « plus petit numéro d'abord », le philosophe 4 (fourchettes 4 et 0) prend
   d'abord la 0. Un cycle d'attente supposerait que chacun attende une fourchette de numéro supérieur
   à celle qu'il tient… ce qui est impossible en fermant le cercle. Pas de cycle, pas d'interblocage.
   C'est **la** technique générale : tout code qui prend plusieurs verrous les prend dans un ordre
   documenté.
3. **Quatre à table** : avec au plus quatre philosophes pour cinq fourchettes, au moins l'un d'eux
   peut toujours obtenir ses deux fourchettes ; il finit, libère, et la file progresse.
4. L'équité n'est **pas** garantie à chaque instant, mais `sync.Mutex` a un **mode famine** : si une
   goroutine attend le verrou depuis plus d'1 ms, le verrou passe en mode équitable (premier arrivé,
   premier servi) jusqu'à ce que la file se vide. Les compteurs de repas restent donc proches, sans
   être égaux.

## Quiz

1. Tout ce qui a été écrit avant `Unlock` est visible par la goroutine qui obtient ensuite le verrou.
2. Pour déverrouiller sur tous les chemins de sortie, panique comprise.
3. Dans la même struct, non exportés, le verrou juste au-dessus des champs qu'il protège.
4. Non : reprendre un verrou qu'on détient bloque pour toujours ; d'où les méthodes `…Locked`.
5. `fatal error: sync: unlock of unlocked mutex`, non récupérable.
6. L'appelant pourrait la lire ou l'écrire sans verrou pendant que d'autres l'écrivent.
7. Deux opérations protégées séparément dont la séquence ne l'est pas ; `-race` ne le voit pas.
8. Quand les lectures sont très majoritaires et réellement concurrentes — à vérifier par mesure.
9. Clés écrites une fois puis beaucoup lues, ou ensembles de clés disjoints entre goroutines.
10. Des E/S, des envois sur channel, des appels à du code inconnu — tout ce qui est lent ou peut bloquer.
