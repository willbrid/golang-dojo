# Leçon 10 — Pathologies

> Un programme concurrent faux ne plante pas toujours. Il peut donner le bon résultat 999 fois
> sur 1 000, se bloquer une fois par semaine en production, ou consommer lentement toute la
> mémoire de la machine. Cette leçon fait l'inventaire des cinq maladies du code concurrent —
> *data race*, interblocage, *livelock*, famine, fuite de goroutines — et des outils qui les
> diagnostiquent, dont le profil **`goroutineleak`** apparu en Go 1.27.

## Objectifs

1. Reconnaître chaque pathologie à ses symptômes.
2. Utiliser le détecteur de races, les traces d'interblocage, `testing/synctest` et le profil `goroutineleak`.
3. Connaître le remède structurel de chacune.

## Explication

### 1. La *data race*

**Définition** : deux goroutines accèdent à la même variable, au moins un accès est une écriture, et
rien ne les ordonne (ni channel, ni verrou, ni atomique, ni `WaitGroup`).

**Symptômes** : résultats faux et variables (compteur qui « perd » des incréments), valeurs
incohérentes (une struct à moitié écrite), et pour les maps : `fatal error: concurrent map writes`.
Pire : le modèle mémoire de Go dit qu'un programme qui contient une race a un comportement
**indéfini** — le compilateur et le processeur peuvent réordonner les accès, et le résultat peut être
n'importe quoi.

**Diagnostic** : le **détecteur de races**, activé par `-race` sur `go test`, `go run`, `go build`.

```
==================
WARNING: DATA RACE
Write at 0x00c00011c018 by goroutine 100018:
  main.main.func1()
      …/main.go:14 +0x…
Previous write at 0x00c00011c018 by goroutine 100017:
  main.main.func1()
      …/main.go:14 +0x…
Goroutine 100018 (running) created at:
  …
==================
```

Il donne les **deux** accès en conflit, avec leurs piles, et où chaque goroutine a été créée. Ses
limites : il ne trouve que les races **qui se produisent pendant l'exécution** observée (d'où des
tests qui exercent la concurrence pour de vrai), il ralentit le programme (×2 à ×20) et multiplie la
mémoire (×5 à ×10) — on l'active en test et en CI, rarement en production. Il ne voit **pas** les
erreurs de logique (« vérifier puis agir », leçon 6).

**Remède** : une seule goroutine propriétaire, ou une synchronisation explicite. Jamais « ça a l'air
de marcher ».

### 2. L'interblocage (*deadlock*)

**Définition** : un cycle de goroutines qui s'attendent mutuellement — A tient le verrou 1 et attend
le 2, B tient le 2 et attend le 1. Ou une goroutine qui attend un channel que personne n'alimentera.

**Symptômes** : si **toutes** les goroutines sont bloquées, le runtime arrête le programme :

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1000 [sync.Mutex.Lock]:
…
```

Mais dans un serveur, d'autres goroutines vivent (réseau, minuteries) : l'interblocage est
**silencieux**. Le service répond de moins en moins, puis plus du tout.

**Diagnostic** : la pile de toutes les goroutines. On l'obtient en envoyant `SIGQUIT` au processus
(Ctrl-\ dans un terminal) : Go affiche toutes les piles et s'arrête. Ou, sans arrêter, par le profil
`goroutine` (`pprof.Lookup("goroutine").WriteTo(os.Stderr, 2)`, ou `/debug/pprof/goroutine?debug=2`
avec `net/http/pprof`, niveau 11). On y cherche des goroutines bloquées depuis longtemps sur
`[sync.Mutex.Lock]`, `[chan send]`, `[chan receive]`, avec la durée (`[chan receive, 42 minutes]`).

**Remèdes** :

- un **ordre global** de prise des verrous (l'exemple et le défi des philosophes, leçon 6) ;
- ne jamais appeler de code inconnu ni envoyer sur un channel **sous verrou** ;
- toute attente a une issue : `select` avec `ctx.Done()`.

### 3. Le *livelock*

**Définition** : les goroutines ne sont pas bloquées — elles **agissent** sans cesse — mais aucune
ne progresse. L'image classique : deux personnes face à face dans un couloir qui s'écartent toutes
les deux du même côté, puis de l'autre, indéfiniment.

**Symptômes** : 100 % de CPU, aucun progrès. Typiquement : des boucles « essayer, échouer,
réessayer immédiatement » qui se gênent mutuellement (des `TryLock` ou des `CompareAndSwap` qui
échouent en boucle, des tentatives réseau synchronisées).

**Remède** : de l'**aléa** dans les réessais (attente aléatoire croissante, *backoff* avec *jitter*,
niveau 10), ou un arbitre qui décide.

### 4. La famine (*starvation*)

**Définition** : une goroutine pourrait progresser mais n'obtient jamais la ressource — parce que
d'autres la reprennent toujours avant elle.

**Exemples** : un écrivain qui n'obtient jamais un `RWMutex` noyé de lecteurs ; un cas de `select`
toujours battu par un autre ; une goroutine qui calcule sans fin sur un `GOMAXPROCS` saturé.

Le runtime limite les dégâts (préemption des goroutines, mode famine du `Mutex` après 1 ms d'attente,
choix aléatoire de `select`, `RWMutex` qui bloque les nouveaux lecteurs quand un écrivain attend).
Mais la conception reste responsable : sections critiques courtes, pas de goroutine qui monopolise.

### 5. La fuite de goroutines

**Définition** : une goroutine lancée qui ne se terminera **jamais**, alors que plus rien n'a besoin
d'elle. Elle garde sa pile et **tout ce qu'elle référence** (buffers, connexions).

**Symptômes** : `runtime.NumGoroutine()` qui ne fait que croître, mémoire qui monte lentement, jusqu'à
l'arrêt du service au bout de jours. C'est la pathologie **la plus fréquente** en production, et la plus
discrète.

**Causes** (toutes vues dans ce niveau) :

- envoi sur un channel non bufferisé que personne ne lit plus (leçons 3 et 4) ;
- réception sur un channel que personne ne fermera ;
- boucle de service sans cas d'arrêt (leçon 4) ;
- étage de pipeline qui n'écoute pas `ctx` (leçon 9).

**Diagnostic, trois outils** :

| Outil | Quand |
|---|---|
| `runtime.NumGoroutine()` et le profil `goroutine` | en production : voir qu'il y a une croissance, et où les goroutines attendent |
| `testing/synctest` | en test : la bulle échoue si une goroutine reste bloquée à la fin (leçon 4) |
| profil **`goroutineleak`** (Go 1.27) | en production ou en test : liste les goroutines **réellement** fuies |

Le profil `goroutineleak` s'appuie sur le ramasse-miettes : une goroutine bloquée sur un channel ou un
verrou que **plus aucune autre goroutine ne peut atteindre** ne pourra jamais être débloquée — elle est
fuie, par définition, et le runtime peut le **prouver**. Le profil liste uniquement celles-là, avec la
ligne où elles sont bloquées. Une goroutine bloquée sur un channel encore référencé ailleurs (qui
*pourrait* être débloquée) n'y figure pas : pas de faux positif. L'inverse n'est pas garanti : le profil
peut **manquer** des fuites réelles (faux négatifs, mesurés au niveau 11, leçon 5) ; son silence ne prouve
donc pas l'absence de fuite, et le nombre de goroutines suivi dans le temps reste indispensable.

```go
pprof.Lookup("goroutineleak").WriteTo(os.Stderr, 1)
```

Il est aussi exposé par `net/http/pprof` (`/debug/pprof/goroutineleak`, niveau 11).

### Le réflexe de prévention

Pour **chaque** `go` : quand se termine-t-elle ? Qui l'attend ? Que se passe-t-il si celui qui devait
lire son résultat est parti ? Si une seule réponse manque, il y a une pathologie en germe.

## Exemple

```go
package main

import (
	"bytes"
	"fmt"
	"path/filepath"
	"runtime"
	"runtime/pprof"
	"strings"
	"sync"
	"time"
)

// leaky abandonne l'attente après un délai, mais la goroutine de calcul reste
// bloquée pour toujours sur son envoi : personne ne lira plus ch.
func leaky() int {
	ch := make(chan int) // le bug : non bufferisé (correction : make(chan int, 1))
	go func() { ch <- compute() }()
	select {
	case v := <-ch:
		return v
	case <-time.After(10 * time.Millisecond):
		return -1
	}
}

func compute() int {
	time.Sleep(50 * time.Millisecond)
	return 42
}

// transfer prend deux verrous. Appelé en sens inverse par deux goroutines,
// il peut s'interbloquer — sauf si les verrous sont pris dans un ORDRE GLOBAL.
type account struct {
	id      int
	mu      sync.Mutex
	balance int
}

func transfer(from, to *account, n int) {
	first, second := from, to
	if to.id < from.id { // ordre global : toujours le plus petit id d'abord
		first, second = to, from
	}
	first.mu.Lock()
	defer first.mu.Unlock()
	second.mu.Lock()
	defer second.mu.Unlock()
	from.balance -= n
	to.balance += n
}

// leakReport rend les fonctions où des goroutines ont fui, d'après le profil goroutineleak (Go 1.27).
func leakReport() (total string, where []string) {
	var buf bytes.Buffer
	pprof.Lookup("goroutineleak").WriteTo(&buf, 1)
	for line := range strings.Lines(buf.String()) {
		switch {
		case strings.HasPrefix(line, "goroutineleak profile:"):
			total = strings.TrimSpace(line)
		case strings.HasPrefix(line, "#") && strings.Contains(line, "main."):
			f := strings.Fields(line) // # adresse fonction+décalage fichier:ligne
			where = append(where, fmt.Sprintf("%s (%s)", strings.Split(f[2], "+")[0], filepath.Base(f[3])))
		}
	}
	return total, where
}

func main() {
	// 1. Fuite de goroutines.
	for range 3 {
		leaky()
	}
	time.Sleep(100 * time.Millisecond)
	fmt.Println("goroutines vivantes :", runtime.NumGoroutine())
	total, where := leakReport()
	fmt.Println(total)
	for _, w := range where {
		fmt.Println("  fuite dans", w)
	}

	// 2. Transferts croisés : pas d'interblocage grâce à l'ordre global.
	a, b := &account{id: 1, balance: 1000}, &account{id: 2, balance: 1000}
	var wg sync.WaitGroup
	for range 1000 {
		wg.Go(func() { transfer(a, b, 1) })
		wg.Go(func() { transfer(b, a, 1) })
	}
	wg.Wait()
	fmt.Println("soldes :", a.balance, b.balance, "— total", a.balance+b.balance)
}
```

Sortie (identique avec `-race` pour la partie « soldes ») :

```
goroutines vivantes : 4
goroutineleak profile: total 3
  fuite dans main.leaky.func1 (main.go:18)
soldes : 1000 1000 — total 2000
```

Et en retirant l'ordre global dans `transfer` :

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1000 [sync.Mutex.Lock]:
…
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `leaky` | Le délai abandonne l'attente ; la goroutine anonyme reste bloquée sur `ch <-` : trois appels, trois fuites. |
| `goroutines vivantes : 4` | `main` + les trois fuites. Le compte dit **qu'il y a** un problème, pas **où**. |
| `goroutineleak profile: total 3` | Le profil prouve que les trois sont définitivement bloquées (plus personne ne peut atteindre `ch`) et donne la **ligne 18** : `go func() { ch <- compute() }()`. |
| La correction de la fuite | `make(chan int, 1)` (leçon 3), ou passer un `ctx` à `compute` (leçon 8). |
| `transfer` et l'ordre par `id` | A→B et B→A prennent les verrous dans le **même** ordre : pas de cycle possible. Sans cette règle, 2 000 transferts croisés s'interbloquent. |
| `total 2000` | L'invariant (la somme des soldes) est préservé : les deux soldes changent sous les deux verrous. |
| `-race` silencieux | Toutes les écritures de `balance` se font sous verrou. |

## Erreurs fréquentes

1. **« Le test passe, donc il n'y a pas de race »** — sans `-race`, ou sans exécution concurrente réelle.
2. **Ignorer un rapport `-race`** « parce que le résultat est bon ».
3. **Des verrous pris dans des ordres différents** selon le chemin.
4. **Réessayer immédiatement en boucle** après un échec : *livelock*.
5. **Des goroutines sans condition de fin**, qu'on découvre en production par la mémoire.
6. **Surveiller la mémoire, pas le nombre de goroutines.**
7. **Croire qu'aucun message `deadlock!` signifie aucun interblocage** : le runtime ne détecte que le
   blocage **total**.

## Bonnes pratiques Go

- `go test -race ./...` en CI, sur tous les paquets, systématiquement.
- `synctest` dans les tests de code concurrent : il détecte les goroutines restantes.
- Exporter `runtime.NumGoroutine()` comme métrique (niveau 12) et alerter sur une croissance continue.
- Exposer le profil `goroutineleak` (et `goroutine`) via `net/http/pprof` sur un port interne.
- Un ordre global documenté pour tout code qui prend plusieurs verrous.
- Réessais avec attente aléatoire croissante.

## Ce que je dois retenir

- **Race** : accès concurrents non ordonnés avec écriture → `-race`.
- **Interblocage** : cycle d'attente → ordre global, pas d'appel inconnu sous verrou, `ctx.Done()`.
- **Livelock** : activité sans progrès → aléa dans les réessais.
- **Famine** : une goroutine jamais servie → sections courtes, équité.
- **Fuite** : goroutine qui ne finit jamais → `NumGoroutine`, `synctest`, profil **`goroutineleak`** (1.27).
- Toute goroutine doit avoir une fin connue.

---

➡️ [Exercices](exercices.md)
