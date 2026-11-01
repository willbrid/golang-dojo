# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Provoquer l'interblocage

```go
// (a)
ch := make(chan int)
ch <- 1                          // goroutine 1 [chan send]

// (b)
a, b := make(chan int), make(chan int)
go func() { <-a; b <- 1 }()      // attend a
<-b                               // main attend b : chacun attend l'autre
                                  // goroutine 1 [chan receive] / goroutine 6 [chan receive]

// (c)
var ch chan int
go func() { ch <- 1 }()
<-ch                              // goroutine 1 [chan receive (nil chan)]
```

La trace indique pour **chaque** goroutine l'opération qui la bloque, et pour un channel nil, la
mention explicite `(nil chan)` : c'est le diagnostic à lire en premier. Sur le cas (c), mesuré :

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive (nil chan)]:
…
goroutine 6 [chan send (nil chan)]:
```

## E2 — Ping-pong

```go
func pingPong(rounds int) []string {
	ping, pong := make(chan int), make(chan int)
	var log []string
	done := make(chan struct{})
	go func() {
		for range rounds {
			n := <-ping
			pong <- n + 1
		}
	}()
	go func() {
		n := 0
		for range rounds {
			ping <- n
			n = <-pong
			log = append(log, fmt.Sprint(n))
		}
		done <- struct{}{}
	}()
	<-done
	return log
}
```

`log` est écrit par une seule goroutine et lu par `main` **après** `<-done` : la réception sur
`done` garantit que tous les `append` sont visibles (modèle mémoire). Pas de race, sortie
`[1 2 3 4 5]`.

## E3 — Direction

```
invalid operation: cannot receive from send-only channel out (variable of type chan<- int)
```

L'erreur arrive **avant** l'exécution, avec la ligne exacte. Le même bug sans direction typée
compilerait, et produirait à l'exécution un interblocage ou une valeur volée à un autre consommateur
— difficile à reproduire, plus encore à localiser.

## E4 — Propriété transférée

`-race` signale une race entre l'écriture de l'émetteur (après l'envoi) et la lecture du
récepteur : le **pointeur** a été envoyé, les deux goroutines partagent désormais le même buffer.
L'envoi ne copie que le pointeur (tableau *Copié ou partagé ?*). Corrections :

- (a) convention de **cession** : après `ch <- buf`, l'émetteur ne touche plus `buf` (il en crée un
  nouveau s'il doit continuer). C'est le style idiomatique ; il repose sur une discipline, que
  `-race` vérifie pendant les tests.
- (b) envoyer `buf.String()` : une **copie** immuable. Plus sûr, au prix d'une allocation.

## E5 — Requête-réponse

```go
type op struct {
	key   string
	incr  bool
	reply chan int
}

func counterServer(ops <-chan op) {
	counts := map[string]int{}   // n'appartient qu'à cette goroutine
	for o := range ops {         // range sur channel : leçon 3 ; un for { o := <-ops … } marche aussi
		if o.incr {
			counts[o.key]++
		}
		o.reply <- counts[o.key]
	}
}
```

Pas de verrou parce qu'**une seule** goroutine accède à la map : toutes les modifications passent
par elle, l'une après l'autre. Le channel **sérialise** les accès. C'est exactement « partager la
mémoire en communiquant ». Le prix : chaque opération coûte deux échanges sur des channels — un
`Mutex` (leçon 6) est plus rapide pour un simple compteur. Le patron se justifie quand l'état est
complexe ou que les opérations doivent être ordonnées.

## Exercice intermédiaire — Un pipeline à trois étages

```go
type line struct {
	text string
	eof  bool
}

func read(r io.Reader, out chan<- line) {
	sc := bufio.NewScanner(r)
	for sc.Scan() {
		out <- line{text: sc.Text()}
	}
	out <- line{eof: true}                 // sentinelle de fin
}

func clean(in <-chan line, out chan<- line) {
	for {
		l := <-in
		if l.eof {
			out <- l                         // propager la fin
			return
		}
		out <- line{text: normalize(l.text)}
	}
}

func count(in <-chan line, result chan<- map[string]int) {
	m := map[string]int{}
	for {
		l := <-in
		if l.eof {
			result <- m
			return
		}
		for _, w := range strings.Fields(l.text) {
			m[w]++
		}
	}
}
```

Défauts de la sentinelle : chaque étage doit la **tester** et la **propager** (oubli = blocage) ;
la valeur de fin mélange donnée et contrôle dans le même type ; l'erreur éventuelle de
`sc.Err()` n'a pas de chemin naturel ; et avec **plusieurs** producteurs, il faudrait compter les
sentinelles. La fermeture de channel (leçon 3) règle les trois premiers points.

Mesure : sur 100 Mo de texte, le pipeline est en général **plus lent** que la boucle séquentielle.
Un échange sur un channel coûte de l'ordre de 100 à 200 ns ; nettoyer une ligne courte coûte du même
ordre. Trois étages ne donnent du parallélisme que si chaque étage a **beaucoup** de travail par
message. Remède classique : envoyer des **lots** (un `[]string` de 1 000 lignes) au lieu de lignes
une à une — la leçon 9 y reviendra.

## Défi — Le coût d'un rendez-vous

1. Un aller-retour ping-pong coûte typiquement **quelques centaines de nanosecondes** ; un appel de
   fonction, une ou deux. L'ordre de grandeur compte plus que le chiffre exact : un échange sur un
   channel est ~100 fois plus cher qu'un appel.
2. Le channel bufferisé de taille 1 n'est pas forcément plus rapide pour un **aller-retour** : chaque
   côté attend quand même la réponse de l'autre ; le buffer aide quand l'émetteur peut prendre de
   l'avance.
3. Avec `GOMAXPROCS=1`, l'échange est souvent **plus rapide** : l'ordonnanceur passe directement
   d'une goroutine à l'autre sur le même P, sans réveiller un autre thread ni faire voyager la donnée
   d'un cache de cœur à l'autre. Hypothèse attendue : le parallélisme a un coût de coordination.
4. Un pipeline à channels devient intéressant quand le travail **par message** est nettement
   supérieur au coût de l'échange — en pratique au-delà de quelques microsecondes (une ligne de
   fichier compressé, une requête réseau, un calcul). Sinon : lots, ou pas de pipeline.

## Quiz

1. L'émetteur bloque jusqu'à l'arrivée d'un récepteur (pour toujours s'il n'y en a jamais).
2. Que le récepteur a reçu la valeur.
3. Tout ce que l'émetteur a écrit avant l'envoi est visible par le récepteur après la réception.
4. Il ne transporte aucune donnée (zéro octet) et dit clairement « événement seulement ».
5. `chan<- T` ne permet que l'envoi ; `<-chan T` que la réception.
6. Quand toutes les goroutines du programme sont bloquées ; pas quand d'autres goroutines vivent
   encore (attente réseau, minuterie…).
7. Elle bloque pour toujours.
8. Pour que chaque appelant reçoive sa propre réponse, sans mélange.
9. Après avoir envoyé un pointeur, l'émetteur ne doit plus lire ni écrire la donnée pointée.
10. Plutôt que de protéger une variable partagée par des verrous, la confier à une goroutine et lui
    envoyer les opérations par channel.
