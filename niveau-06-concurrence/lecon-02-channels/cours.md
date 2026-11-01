# Leçon 2 — Channels non bufferisés

> *« Ne communiquez pas en partageant la mémoire ; partagez la mémoire en communiquant. »*
> Ce proverbe de Go résume l'idée des **channels** : plutôt que plusieurs goroutines qui lisent
> et écrivent la même variable sous verrou, une goroutine **envoie** une valeur à une autre, qui
> en devient responsable. Cette leçon commence par la forme la plus simple — et la plus
> contraignante — : le channel sans buffer, où chaque échange est un **rendez-vous**.

## Objectifs

1. Créer un channel, envoyer, recevoir, et comprendre le rendez-vous.
2. Utiliser un channel comme **signal** de synchronisation.
3. Typer la **direction** d'un channel dans les signatures.
4. Reconnaître l'interblocage le plus simple et son message.

## Explication

### Créer, envoyer, recevoir

```go
ch := make(chan int)   // channel non bufferisé d'int
ch <- 42               // envoyer
v := <-ch              // recevoir
<-ch                   // recevoir en ignorant la valeur
```

Un channel est **typé** : `chan int` ne transporte que des `int`. Sa zéro-valeur est `nil` (voir
plus bas) : il faut `make`. Comme une map, un channel est une **référence** vers une structure
interne : le copier (le passer à une fonction) donne une seconde poignée sur le **même** channel —
c'est la ligne « channel » du tableau *Copié ou partagé ?*.

### Le rendez-vous

Sur un channel **non bufferisé** :

- un **envoi** bloque jusqu'à ce qu'un récepteur prenne la valeur ;
- une **réception** bloque jusqu'à ce qu'un émetteur en fournisse une.

Les deux goroutines doivent être présentes **au même moment** : c'est un rendez-vous. D'où une
propriété précieuse : quand l'envoi se termine, l'émetteur **sait** que le récepteur a la valeur.
L'échange transfère la donnée **et** synchronise les deux goroutines.

Cette synchronisation a une conséquence formelle (le **modèle mémoire** de Go) : tout ce que
l'émetteur a écrit **avant** l'envoi est visible par le récepteur **après** la réception. C'est ce
qui permet de passer un pointeur vers une grosse structure par un channel, sans verrou : après
l'échange, le récepteur peut la lire — à condition que l'émetteur ne la modifie plus.

### Un channel comme signal

Quand seule la synchronisation compte, on envoie une valeur vide :

```go
done := make(chan struct{})
go func() {
	travailler()
	done <- struct{}{}    // « j'ai fini »
}()
<-done                     // attendre
```

`struct{}` ne pèse **zéro octet** : la convention dit « ce channel ne transporte pas de donnée,
seulement un événement ».

### La direction d'un channel

Un type de channel peut être **restreint** à un sens :

| Type | Peut | Ne peut pas |
|---|---|---|
| `chan T` | envoyer et recevoir | |
| `chan<- T` | envoyer seulement | recevoir |
| `<-chan T` | recevoir seulement | envoyer |

```go
func produce(out chan<- string)   // produce ne fait qu'écrire
func consume(in <-chan string)    // consume ne fait que lire
```

Un `chan T` se convertit **implicitement** en l'une ou l'autre forme à l'appel. L'erreur est
détectée à la compilation :

```
invalid operation: cannot send to receive-only channel <-chan int in
```

Typer la direction **documente** le rôle de chaque fonction et empêche une classe d'erreurs.
C'est la règle dans tout code idiomatique.

### L'interblocage le plus simple

```go
func main() {
	ch := make(chan int)
	ch <- 1          // personne ne recevra jamais
}
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	…/main.go:2 +0x28
```

Le runtime détecte que **toutes** les goroutines sont bloquées et arrête le programme. La pile
indique sur quelle opération chacune attend (`chan send`, `chan receive`). Attention : cette
détection ne fonctionne que si **toutes** les goroutines sont bloquées. Dans un serveur, où des
goroutines attendent le réseau, deux goroutines interbloquées restent bloquées **silencieusement**
pour toujours — c'est une fuite (leçon 10).

### Le channel nil

Envoyer ou recevoir sur un channel **nil** bloque **pour toujours**. Ce n'est pas une erreur — on
s'en servira volontairement avec `select` (leçon 4) pour désactiver un cas — mais un `var ch chan
int` oublié sans `make` produit un blocage incompréhensible.

### Le motif de la requête-réponse

Un channel peut transporter… un channel. Une requête contient son propre channel de réponse :

```go
type request struct {
	n     int
	reply chan int
}
```

Le serveur reçoit la requête, calcule, répond **sur le channel de la requête**. Chaque appelant
reçoit sa propre réponse, sans confusion possible. C'est le principe de nombreux services internes
écrits en Go, et de l'API de `net/http` sous le capot.

## Exemple

```go
package main

import (
	"fmt"
	"strings"
	"time"
)

// stage lit ses entrées sur in, écrit ses sorties sur out.
// Les types de paramètres FIXENT le sens : stage ne peut pas écrire sur in.
func upper(n int, in <-chan string, out chan<- string) {
	for range n {
		s := <-in
		out <- strings.ToUpper(s)
	}
}

func main() {
	// 1. Le rendez-vous : l'envoi attend le récepteur.
	ch := make(chan string)
	waited := make(chan time.Duration)
	go func() {
		start := time.Now()
		ch <- "ping" // bloque jusqu'à ce que main reçoive
		waited <- time.Since(start)
	}()
	time.Sleep(100 * time.Millisecond) // main fait exprès d'arriver en retard au rendez-vous
	fmt.Println("reçu :", <-ch)
	fmt.Println("l'émetteur a attendu", (<-waited).Round(10*time.Millisecond))

	// 2. Un channel comme signal de fin : remplace un WaitGroup pour UNE goroutine.
	done := make(chan struct{})
	go func() {
		fmt.Println("travail…")
		done <- struct{}{}
	}()
	<-done

	// 3. Une chaîne de goroutines reliées par des channels.
	words := []string{"go", "channel", "rendez-vous"}
	in, out := make(chan string), make(chan string)
	go upper(len(words), in, out)
	go func() {
		for _, w := range words {
			in <- w
		}
	}()
	for range words {
		fmt.Println(<-out)
	}

	// 4. La réponse dans le channel : une requête porte son propre channel de réponse.
	type request struct {
		n     int
		reply chan int
	}
	reqs := make(chan request)
	go func() { // un « serveur » qui calcule des carrés
		for {
			r := <-reqs
			r.reply <- r.n * r.n
		}
	}()
	for _, n := range []int{3, 4} {
		r := request{n: n, reply: make(chan int)}
		reqs <- r
		fmt.Println(n, "² =", <-r.reply)
	}
}
```

Sortie :

```
reçu : ping
l'émetteur a attendu 100ms
travail…
GO
CHANNEL
RENDEZ-VOUS
3 ² = 9
4 ² = 16
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `ch <- "ping"` qui attend 100 ms | Le rendez-vous rendu visible : l'émetteur reste bloqué tant que `main` n'est pas venu recevoir. |
| `waited` | La goroutine **envoie** sa mesure au lieu de l'afficher : l'ordre d'affichage est fixé par `main`, donc déterministe. Deux goroutines qui affichent chacune de leur côté donneraient un ordre quelconque. |
| `time.Sleep` dans `main` | Seulement pour la démonstration : il crée le retard qu'on veut observer. Jamais comme moyen de synchronisation. |
| `done chan struct{}` | Un signal de fin pour **une** goroutine. Pour plusieurs, `WaitGroup` (leçon 5). |
| `upper(n, in <-chan, out chan<-)` | Les directions dans la signature : `upper` ne peut pas écrire sur `in` ni lire `out`. |
| `for range n` dans `upper` | Il faut savoir **combien** de valeurs arrivent. La leçon 3 remplacera ce compte par la **fermeture** du channel. |
| Le producteur dans sa propre goroutine | Si `main` envoyait les trois mots lui-même avant de lire `out`, `upper` resterait bloqué sur son premier envoi vers `out`, et `main` sur son deuxième envoi vers `in` : interblocage. |
| `request{reply: make(chan int)}` | Chaque requête a son canal de réponse. |
| Le serveur `for { … }` | Il ne se termine jamais : ici sans conséquence (le programme s'arrête avec `main`), mais dans un vrai programme, c'est une **fuite**. La leçon 3 (fermeture) et la leçon 8 (`context`) apprendront à l'arrêter. |

## Erreurs fréquentes

1. **Envoyer et recevoir dans la même goroutine** sur un channel non bufferisé : interblocage.
2. **Oublier `make`** : un channel nil bloque pour toujours.
3. **Ne pas typer la direction** dans les signatures.
4. **Afficher depuis plusieurs goroutines** et s'étonner de l'ordre.
5. **Supposer qu'un envoi réussit sans récepteur.**
6. **Lancer une goroutine-serveur sans moyen de l'arrêter.**
7. **Modifier une donnée après l'avoir envoyée** par pointeur : le récepteur et l'émetteur la
   partagent désormais.

## Bonnes pratiques Go

- Typer la direction : `chan<- T` pour produire, `<-chan T` pour consommer.
- `chan struct{}` pour un signal sans donnée.
- Après avoir envoyé un pointeur, considérer qu'on en a **cédé la propriété**.
- Chaque goroutine qui envoie ou reçoit doit avoir un partenaire garanti, ou un moyen d'abandonner.
- Un seul endroit pour afficher : les goroutines **communiquent** leurs résultats.

## Ce que je dois retenir

- `make(chan T)` ; `ch <- v` envoie, `v := <-ch` reçoit.
- Non bufferisé = **rendez-vous** : l'envoi attend le récepteur, et l'échange synchronise.
- Ce qui est écrit avant l'envoi est visible après la réception.
- `chan<- T` / `<-chan T` : direction vérifiée à la compilation.
- `all goroutines are asleep - deadlock!` : tout le programme est bloqué.
- Un channel nil bloque pour toujours.

---

🧵 **Fil rouge :** la ligne « channel » du tableau *[Copié ou partagé ?](../../ressources/copie-ou-partage.md)* prend maintenant tout son sens.

➡️ [Exercices](exercices.md)
