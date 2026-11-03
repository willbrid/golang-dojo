# Module 3 — Systèmes distribués

> Dès qu'un service a deux instances et une base répliquée, il devient un système distribué, que ses auteurs
> l'aient voulu ou non. Plusieurs certitudes disparaissent alors :
> - un appel peut avoir réussi sans qu'on le sache ;
> - deux horloges ne donnent pas la même heure ;
> - une réplique peut être en retard ;
> - le réseau peut couper le système en deux.
>
> Ce module ne réimplémente pas un algorithme de consensus : on n'en écrit jamais soi-même. Il donne les
> raisonnements qui évitent les erreurs classiques, et il **mesure** chacune d'elles par une simulation
> déterministe : écritures perdues à cause du décalage des horloges, lectures périmées selon les quorums,
> écrivain fantôme d'un verrou à bail, divergence pendant une partition.

## Objectifs

- Raisonner sur les pannes partielles : un délai expiré ne dit pas si l'opération a eu lieu.
- Savoir pourquoi l'heure murale ne sert pas à ordonner des événements entre machines, et ce qui la
  remplace.
- Choisir des quorums de lecture et d'écriture d'après la cohérence voulue.
- Comprendre le choix imposé par une partition : refuser ou diverger.
- Savoir ce que garantit un service de consensus, et protéger une ressource par un jeton de clôture.

## Explication

### Ce qui change avec plusieurs machines

Sur une seule machine, un appel de fonction réussit, échoue ou panique, et on le sait. Entre deux machines,
un appel a **trois** issues : réussi, échoué, ou **inconnu**. Un délai dépassé ne dit pas si le serveur a
traité la requête avant que la réponse ne se perde. Il faut donc :

- **des opérations idempotentes**, ou une clé d'idempotence (N10 L9), pour pouvoir réessayer sans dupliquer ;
- **accepter qu'« exactement une fois » n'existe pas** en transport. On a « au moins une fois » (réessayer)
  plus un traitement idempotent (dédoublonner), c'est le patron *outbox* et le consommateur idempotent de
  N10 L8 ;
- **des pannes partielles** : une instance répond, l'autre non, la base est lente pour une seule.

Les messages peuvent être perdus, dupliqués, retardés, réordonnés. Un composant qui suppose le contraire
fonctionne en test, sur une machine, et casse en production.

### L'heure murale ne sert pas à ordonner

Les horloges de deux machines diffèrent de quelques millisecondes, parfois de secondes, malgré la
synchronisation NTP. Une horloge peut aussi **reculer** quand elle est corrigée. En Go, `time.Now()` contient
une lecture **monotone** qui ne recule jamais. Elle ne vaut que dans le processus et n'est pas transmise par
`MarshalJSON` ni par `Format` : elle ne sert à rien d'une machine à l'autre.

La stratégie « le dernier écrit gagne », qui garde la valeur au plus grand horodatage, **perd des écritures**
dès que les horloges diffèrent. Mesuré dans l'exemple : deux écritures successives, séparées en temps réel
de 0 à 10 ms, la seconde sur une machine dont l'horloge retarde.

| Retard de l'horloge du second écrivain | Écritures plus récentes perdues |
|---|---|
| 0 | 0 % |
| 1 ms | 9,9 % |
| 5 ms | 49,9 % |
| 20 ms | **100 %** |

Avec 20 ms de décalage, ce qui reste raisonnable pour des machines mal synchronisées, **toutes** les
écritures rapprochées sont perdues, sans aucune erreur.

Une **horloge logique** (horloge de Lamport) ordonne les événements qui ont un lien de **cause** :
- chaque nœud incrémente un compteur à chaque événement ;
- il l'envoie avec ses messages ;
- en recevant, il prend le maximum des deux, plus un.

Si l'écriture B a été faite **après avoir lu** A, l'horodatage de B dépasse celui de A, quelles que soient les
horloges : 0 % de perte mesurée. Pour deux écritures **concurrentes**, qui ne se sont pas vues, aucune horloge
ne dit laquelle est « la bonne ». C'est un **conflit**, à résoudre explicitement :
- un numéro de version vérifié à l'écriture (verrouillage optimiste, N8 L4) ;
- une fusion propre au métier ;
- des structures de données faites pour se fusionner.

### Réplication et quorums

Une donnée copiée sur N répliques : une écriture est confirmée quand **W** répliques l'ont reçue, une lecture
interroge **R** répliques et garde la version la plus récente. Si **W + R > N**, tout ensemble de lecture
rencontre au moins une réplique qui a l'écriture, et la lecture n'est **jamais** périmée. Mesuré sur 3
répliques (les autres reçoivent l'écriture avec un retard de 0 à 20 ms, la lecture part juste après la
confirmation) :

| W | R | W + R > 3 ? | Lectures périmées |
|---|---|---|---|
| 1 | 1 | non | 66,6 % |
| 1 | 2 | non | 33,1 % |
| 2 | 1 | non | 33,5 % |
| 2 | 2 | **oui** | **0 %** |
| 3 | 1 | **oui** | **0 %** |
| 1 | 3 | **oui** | **0 %** |
| 1 | 1, lecture 30 ms après | non | 0 % (tout est arrivé) |

Le choix de W et R est un arbitrage :

- **W = N** : des lectures rapides et à jour, mais une seule réplique en panne bloque les écritures.
- **W = R = 2 sur 3** : une panne tolérée, à la fois pour lire et pour écrire.
- **W = R = 1** : rapide et toujours disponible, mais une lecture juste après une écriture est périmée deux
  fois sur trois.

La dernière ligne montre la **cohérence à terme** (*eventual consistency*) : sans nouvelle écriture, toutes
les répliques finissent par converger, mais personne ne dit quand. « Lire ses propres écritures » se règle
autrement : lire sur le leader, ou sur une réplique qui a au moins la version qu'on vient d'écrire.

Les principaux modèles, du plus fort au plus faible :

| Modèle | Garantie | Exemple |
|---|---|---|
| linéarisable | chaque opération semble instantanée, dans l'ordre réel (module 1) | lecture sur le leader d'un système de consensus |
| séquentiel | un ordre global, compatible avec l'ordre de chaque client | — |
| causal | ce qui a pu causer un événement est vu avant lui | horloges logiques |
| à terme | les répliques finissent par converger | réplication asynchrone, caches |

### Partitions : refuser ou diverger

Une **partition** coupe le réseau : le nœud 0 ne joint plus les nœuds 1 et 2, qui se joignent entre eux. Les
clients du côté du nœud 0 veulent écrire. Il n'y a que deux choix, mesurés dans l'exemple sur 1 000 écritures
de chaque côté :

| Politique | Acceptées côté majoritaire | Acceptées côté minoritaire | Refusées | Conflits à la réparation |
|---|---|---|---|---|
| quorum majoritaire (2 sur 3) | 1 000 | 0 | 1 000 | 0 |
| chacun accepte localement | 1 000 | 1 000 | 0 | **1 000** |

- **Refuser**, côté minoritaire, garde la cohérence : il n'y a qu'une histoire. Mais les clients de ce côté
  reçoivent des erreurs pendant toute la partition.
- **Accepter partout** garde la disponibilité, mais crée deux histoires. À la réparation, 1 000 clés ont deux
  valeurs, et il faut une règle pour choisir. C'est souvent « le dernier écrit gagne », avec les pertes de la
  section sur les horloges.

C'est le théorème dit « CAP » : pendant une partition, on choisit entre cohérence et disponibilité. Hors
partition, l'arbitrage continue entre cohérence et **latence** : attendre W répliques prend du temps. Ce choix
dépend du métier : un panier d'achat peut diverger, un solde bancaire non.

### Le consensus, et ce qu'on en fait

Le **consensus** fait décider à plusieurs nœuds une même suite de valeurs (un journal), même quand certains
tombent ou que le réseau se coupe, tant qu'une **majorité** se joint. Les algorithmes de référence, Raft et
Paxos, reposent sur une propriété : deux majorités d'un même ensemble ont toujours un nœud en commun. Une
décision prise par une majorité ne peut donc pas être contredite par une autre.

**On ne les implémente pas.** On les utilise à travers un système éprouvé : etcd, Consul, ZooKeeper, ou une
base qui l'intègre. On s'en sert pour l'élection d'un leader, un verrou réparti, une configuration partagée,
ou l'attribution de numéros uniques.

Le piège le plus courant est le **verrou à bail**. Un service accorde un verrou pour 100 ms renouvelables ; le
détenteur doit le renouveler avant l'échéance. Si le détenteur A s'arrête 150 ms, le bail expire et B obtient
le verrou. Les causes possibles : une pause du GC, une machine virtuelle suspendue, un réseau lent. Quand A se
réveille, il **croit** toujours tenir le verrou, et écrit :

| Ressource protégée | Écritures acceptées | L'écriture tardive de A |
|---|---|---|
| sans jeton | `[B A]` | **acceptée** : A écrase le travail de B |
| avec jeton de clôture | `[B]` | refusée |

Le **jeton de clôture** (*fencing token*) est un nombre qui croît à chaque attribution du verrou. Le
service de verrou le donne avec le bail (etcd fournit un numéro de révision qui joue ce rôle), et l'écrivain
le présente à la ressource. La ressource refuse tout jeton plus petit que le dernier vu. La vérification est
faite par la ressource, pas par le client : le client ne peut pas savoir qu'il a été mis en pause.

### Tester un système distribué

Les défauts d'un système distribué apparaissent sous des ordres d'événements rares. On les cherche par :

- **la simulation déterministe**, comme dans l'exemple : temps virtuel, réseau simulé, graine fixée. Un
  échec se rejoue à l'identique ;
- **l'injection de pannes** dans un vrai déploiement : coupures réseau, latences, horloges décalées,
  processus tués ;
- **la vérification des historiques** : la linéarisabilité du module 1 (défi), sur les opérations observées
  par les clients.

## Exemple

Quatre simulations en temps virtuel : aucune horloge réelle ni aucun réseau, donc des résultats identiques à
chaque exécution.

```go.mod
// Fichier : go.mod
module example.com/dist

go 1.27
```

```go
// Fichier : dist/lww.go
// Package dist : trois expériences sur des systèmes répartis, simulées en temps virtuel (déterministes).
package dist

import (
	"math/rand/v2"
	"time"
)

// --- 1. « Le dernier écrit gagne » avec des horloges décalées ---

// LostWrites simule n paires d'écritures sur un registre répliqué qui garde la valeur au plus grand
// horodatage. Deux clients écrivent, l'un après l'autre en temps RÉEL (écart aléatoire de 0 à gap) ;
// l'horloge du second est en retard de skew. Renvoie la fraction d'écritures plus récentes perdues.
func LostWrites(seed uint64, n int, gap, skew time.Duration) float64 {
	rng := rand.New(rand.NewPCG(seed, 1))
	lost := 0
	for range n {
		realFirst := time.Duration(rng.Int64N(int64(time.Hour)))
		realSecond := realFirst + time.Duration(rng.Int64N(int64(gap)))
		tsFirst := realFirst          // horloge exacte
		tsSecond := realSecond - skew // horloge en retard
		if tsSecond <= tsFirst {      // le registre garde la PREMIÈRE : l'écriture réellement plus récente est perdue
			lost++
		}
	}
	return float64(lost) / float64(n)
}

// Lamport : une horloge logique. Chaque nœud avance son compteur à chaque événement, et prend le
// maximum de son compteur et de celui reçu, plus un, à chaque message.
type Lamport struct{ t uint64 }

func (l *Lamport) Tick() uint64              { l.t++; return l.t }
func (l *Lamport) Recv(remote uint64) uint64 { l.t = max(l.t, remote) + 1; return l.t }

// LostWritesLamport : le même scénario, mais le second client a LU la valeur du premier avant
// d'écrire (il en a donc reçu l'horodatage logique). Une écriture qui « a vu » l'autre la dépasse.
func LostWritesLamport(seed uint64, n int) float64 {
	lost := 0
	for range n {
		var a, b Lamport
		tsFirst := a.Tick()
		b.Recv(tsFirst) // le second client lit la valeur écrite (et son horodatage)
		tsSecond := b.Tick()
		if tsSecond <= tsFirst {
			lost++
		}
	}
	return float64(lost) / float64(n)
}
```

```go
// Fichier : dist/quorum.go
package dist

import (
	"math/rand/v2"
	"time"
)

// --- 2. Réplication et quorums ---

// StaleReads simule n cycles « écrire v, puis lire tout de suite » sur N répliques. Une écriture est
// acquittée quand W répliques l'ont reçue ; les autres la reçoivent avec un délai aléatoire (0..lag).
// La lecture interroge R répliques choisies au hasard et garde la version la plus récente.
// Renvoie la fraction de lectures qui ne voient pas l'écriture qu'elles suivent.
func StaleReads(seed uint64, n, N, W, R int, lag, readAfter time.Duration) float64 {
	rng := rand.New(rand.NewPCG(seed, 2))
	stale := 0
	for range n {
		// Instant de réception de l'écriture par chaque réplique, relatif à l'envoi.
		arrive := make([]time.Duration, N)
		for i := range arrive {
			arrive[i] = time.Duration(rng.Int64N(int64(lag)))
		}
		// L'acquittement arrive quand W répliques l'ont : le W-ième plus petit délai.
		sorted := append([]time.Duration(nil), arrive...)
		for i := range sorted { // tri par insertion, N est petit
			for j := i; j > 0 && sorted[j] < sorted[j-1]; j-- {
				sorted[j], sorted[j-1] = sorted[j-1], sorted[j]
			}
		}
		ack := sorted[W-1]
		readAt := ack + readAfter // la lecture part après l'acquittement
		seen := false
		for _, i := range rng.Perm(N)[:R] {
			if arrive[i] <= readAt {
				seen = true
			}
		}
		if !seen {
			stale++
		}
	}
	return float64(stale) / float64(n)
}
```

```go
// Fichier : dist/fencing.go
package dist

import "time"

// --- 3. Le verrou à bail, et le jeton de clôture ---

// LockService accorde un bail exclusif d'une durée fixe, avec un jeton croissant à chaque attribution.
type LockService struct {
	holder  string
	expires time.Duration
	token   uint64
	ttl     time.Duration
}

// Acquire accorde le verrou si personne ne le tient (ou si le bail a expiré) à l'instant now.
func (l *LockService) Acquire(who string, now time.Duration) (uint64, bool) {
	if l.holder != "" && now < l.expires {
		return 0, false
	}
	l.holder, l.expires = who, now+l.ttl
	l.token++
	return l.token, true
}

// Storage : la ressource protégée. Avec fencing, elle refuse un jeton plus ancien que le dernier vu.
type Storage struct {
	Fencing   bool
	lastToken uint64
	Writes    []string
}

func (s *Storage) Write(who string, token uint64) bool {
	if s.Fencing && token < s.lastToken {
		return false // jeton périmé : l'écrivain a perdu le verrou sans le savoir
	}
	s.lastToken = max(s.lastToken, token)
	s.Writes = append(s.Writes, who)
	return true
}

// PausedClient rejoue le scénario classique : A obtient le verrou, s'arrête (pause du GC, machine
// virtuelle suspendue, réseau lent) plus longtemps que le bail ; B obtient le verrou et écrit ;
// A se réveille, CROIT toujours tenir le verrou, et écrit à son tour.
func PausedClient(fencing bool, ttl, pause time.Duration) (writes []string, rejected bool) {
	lock := &LockService{ttl: ttl}
	st := &Storage{Fencing: fencing}
	tokA, _ := lock.Acquire("A", 0)
	// A est en pause de 0 à pause.
	tokB, okB := lock.Acquire("B", ttl+time.Millisecond)
	if okB {
		st.Write("B", tokB)
	}
	okA := st.Write("A", tokA) // A, à son réveil, à l'instant pause
	return st.Writes, !okA
}
```

```go
// Fichier : dist/partition.go
package dist

// --- 4. Partition : refuser ou diverger ---

// PartitionResult décrit ce qui s'est passé pendant une partition {nœud 0} | {nœuds 1, 2}.
type PartitionResult struct {
	AcceptedMajority, AcceptedMinority, Rejected int
	ConflictsAtHeal                              int // clés écrites des deux côtés, à réconcilier
}

// Partition : des clients de chaque côté écrivent writesPerSide clés (les mêmes clés des deux côtés).
// quorum : une écriture exige 2 nœuds sur 3 (le côté minoritaire refuse) ; sinon, chaque côté accepte
// localement et réplique plus tard.
func Partition(quorum bool, writesPerSide int) PartitionResult {
	var r PartitionResult
	minority := map[int]bool{}
	majority := map[int]bool{}
	for k := range writesPerSide {
		majority[k] = true // le côté {1, 2} a 2 nœuds : il atteint toujours le quorum
		r.AcceptedMajority++
		if quorum {
			r.Rejected++ // le nœud 0, seul, ne peut pas réunir 2 nœuds : il refuse
		} else {
			minority[k] = true
			r.AcceptedMinority++
		}
	}
	for k := range minority {
		if majority[k] {
			r.ConflictsAtHeal++ // deux valeurs pour la même clé : qui gagne ?
		}
	}
	return r
}
```

```go
// Fichier : dist/dist_test.go
package dist

import (
	"fmt"
	"testing"
	"time"
)

func TestLWW(t *testing.T) {
	for _, skew := range []time.Duration{0, time.Millisecond, 5 * time.Millisecond, 20 * time.Millisecond, 100 * time.Millisecond} {
		fmt.Printf("écarts réels 0–10 ms, horloge en retard de %-6v : %5.1f %% d'écritures récentes perdues\n",
			skew, 100*LostWrites(1, 100_000, 10*time.Millisecond, skew))
	}
	fmt.Printf("horloges de Lamport, écriture qui a lu la précédente : %5.1f %% perdues\n", 100*LostWritesLamport(1, 100_000))
}

func TestQuorum(t *testing.T) {
	const lag = 20 * time.Millisecond
	for _, c := range []struct{ W, R int }{{1, 1}, {1, 2}, {2, 1}, {2, 2}, {3, 1}, {1, 3}} {
		fmt.Printf("N=3 W=%d R=%d (W+R %s N) : %5.1f %% de lectures périmées\n", c.W, c.R,
			map[bool]string{true: ">", false: "≤"}[c.W+c.R > 3], 100*StaleReads(1, 100_000, 3, c.W, c.R, lag, 0))
	}
	fmt.Printf("N=3 W=1 R=1, lecture 30 ms après l'acquittement : %5.1f %%\n", 100*StaleReads(1, 100_000, 3, 1, 1, lag, 30*time.Millisecond))
}

func TestFencing(t *testing.T) {
	for _, f := range []bool{false, true} {
		w, rej := PausedClient(f, 100*time.Millisecond, 150*time.Millisecond)
		fmt.Printf("jeton de clôture=%-5v écritures acceptées %v, écriture de A refusée : %v\n", f, w, rej)
	}
}

func TestPartition(t *testing.T) {
	for _, q := range []bool{true, false} {
		r := Partition(q, 1000)
		fmt.Printf("quorum=%-5v acceptées côté majoritaire %4d, côté minoritaire %4d, refusées %4d, conflits à la réparation %4d\n",
			q, r.AcceptedMajority, r.AcceptedMinority, r.Rejected, r.ConflictsAtHeal)
	}
}
```

```
$ go test -v ./dist/
écarts réels 0–10 ms, horloge en retard de 0s     :   0.0 % d'écritures récentes perdues
écarts réels 0–10 ms, horloge en retard de 1ms    :   9.9 % d'écritures récentes perdues
écarts réels 0–10 ms, horloge en retard de 5ms    :  49.9 % d'écritures récentes perdues
écarts réels 0–10 ms, horloge en retard de 20ms   : 100.0 % d'écritures récentes perdues
écarts réels 0–10 ms, horloge en retard de 100ms  : 100.0 % d'écritures récentes perdues
horloges de Lamport, écriture qui a lu la précédente :   0.0 % perdues
N=3 W=1 R=1 (W+R ≤ N) :  66.6 % de lectures périmées
N=3 W=1 R=2 (W+R ≤ N) :  33.1 % de lectures périmées
N=3 W=2 R=1 (W+R ≤ N) :  33.5 % de lectures périmées
N=3 W=2 R=2 (W+R > N) :   0.0 % de lectures périmées
N=3 W=3 R=1 (W+R > N) :   0.0 % de lectures périmées
N=3 W=1 R=3 (W+R > N) :   0.0 % de lectures périmées
N=3 W=1 R=1, lecture 30 ms après l'acquittement :   0.0 %
jeton de clôture=false écritures acceptées [B A], écriture de A refusée : false
jeton de clôture=true  écritures acceptées [B], écriture de A refusée : true
quorum=true  acceptées côté majoritaire 1000, côté minoritaire    0, refusées 1000, conflits à la réparation    0
quorum=false acceptées côté majoritaire 1000, côté minoritaire 1000, refusées    0, conflits à la réparation 1000
```

## Explication du code

| Élément | Rôle |
|---|---|
| `LostWrites` | Les instants **réels** sont tirés au hasard ; seuls les **horodatages** portent le décalage. Le registre garde le plus grand horodatage, comme une base en « dernier écrit gagne ». |
| `Lamport.Recv` | `max(local, reçu) + 1` : l'horodatage d'un événement dépasse celui de tout ce qu'il a vu. |
| `StaleReads` | Chaque réplique reçoit l'écriture avec un délai aléatoire. La confirmation arrive au W-ième délai, et la lecture est périmée si aucune des R répliques lues n'a encore l'écriture. |
| `rng.Perm(N)[:R]` | R répliques distinctes, choisies au hasard. |
| `LockService.Acquire` | Un bail à durée fixe, et un jeton incrémenté à **chaque** attribution. |
| `Storage.Write` | La vérification du jeton est faite **par la ressource**, la seule à voir toutes les écritures. |
| `PausedClient` | A obtient le verrou à 0, B à 101 ms (le bail de A a expiré), A écrit à son réveil, à 150 ms. |
| `Partition` | Le côté minoritaire, seul, ne réunit jamais 2 nœuds sur 3. |

## Erreurs fréquentes

1. **Traiter un délai expiré comme un échec** et réessayer une opération non idempotente : débit en double,
   courriel envoyé deux fois.
2. **Ordonner des écritures par l'heure de machines différentes.**
3. **Envoyer `time.Now()` d'une machine à l'autre** en croyant transmettre une heure monotone.
4. **Lire sur une réplique asynchrone juste après avoir écrit**, et s'étonner de ne pas voir sa propre
   écriture.
5. **Un verrou à bail sans jeton de clôture** : deux écrivains en même temps dès la première pause du GC.
6. **Écrire son propre algorithme de consensus** ou d'élection de leader.
7. **Choisir la disponibilité pendant une partition** sans avoir écrit la règle de réconciliation.
8. **Tester un protocole réparti seulement sur un réseau local parfait.**

## Bonnes pratiques Go

- Clés d'idempotence et réessais bornés pour tout appel distant (N10 L9) ; « au moins une fois » plus
  dédoublonnage.
- Horloge monotone (`time.Since`) pour les durées dans un processus ; numéros de version ou horloges logiques
  entre machines.
- W + R > N quand la lecture doit voir la dernière écriture ; sinon, documenter le retard accepté.
- Un service de consensus existant (etcd, ou la base) pour les leaders et les verrous, toujours avec un jeton
  de clôture vérifié par la ressource.
- La politique de partition (refuser ou diverger, et comment réconcilier) écrite avant le code, et testée par
  simulation.
- Des simulations déterministes avec graine fixée pour les protocoles ; de l'injection de pannes pour le
  système réel.

## Ce que je dois retenir

- Un appel distant a trois issues : réussi, échoué, inconnu. L'idempotence rend le réessai sûr.
- L'heure murale perd des écritures : 100 % avec 20 ms de décalage. Les horloges logiques ordonnent ce qui a un
  lien de cause.
- W + R > N supprime les lectures périmées (66,6 % → 0 %).
- Pendant une partition : refuser (cohérence) ou diverger (disponibilité), et réconcilier.
- Un verrou à bail protège mal ; le jeton de clôture vérifié par la ressource protège.

---

➡️ [Exercices](exercices.md)
