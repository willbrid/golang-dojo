# Leçon 3 — Channels bufferisés et fermeture

> La leçon 2 laissait deux questions ouvertes : comment un consommateur sait-il que plus rien
> n'arrivera, et comment un producteur peut-il prendre de l'avance sur son consommateur ? Les
> réponses sont la **fermeture** d'un channel et le **buffer**. Avec elles vient la règle qui
> évite la plupart des paniques en code concurrent : **c'est l'émetteur qui ferme**.

## Objectifs

1. Créer un channel bufferisé et comprendre quand il bloque.
2. Fermer un channel, détecter la fermeture avec `v, ok` et `range`.
3. Appliquer la règle « l'émetteur ferme », y compris avec plusieurs émetteurs.
4. Choisir une capacité de buffer — souvent 0 ou 1, rarement plus.

## Explication

### Le buffer

```go
jobs := make(chan int, 3)   // capacité 3
```

Un channel bufferisé contient une **file** de taille fixe :

- un **envoi** bloque seulement si le buffer est **plein** ;
- une **réception** bloque seulement si le buffer est **vide**.

`len(ch)` donne le nombre d'éléments en attente, `cap(ch)` la capacité. Ces valeurs ne servent
qu'au diagnostic : dans un programme concurrent, `len(ch)` est **périmée** dès qu'on l'a lue — ne
jamais écrire `if len(ch) < cap(ch) { ch <- v }` en croyant éviter un blocage.

Un buffer **découple** le rythme de l'émetteur de celui du récepteur, mais n'en change pas le débit
moyen : si le consommateur est durablement plus lent, le buffer se remplit, puis l'émetteur bloque
comme avant. Un buffer absorbe des **à-coups**, il ne résout pas une lenteur. La leçon 11
(*backpressure*) en fait le cœur de la conception.

Ce que le buffer **perd** : la garantie du rendez-vous. Quand un envoi sur un channel bufferisé se
termine, l'émetteur ne sait **pas** si quelqu'un a reçu la valeur.

### Fermer

```go
close(ch)
```

Fermer signifie « **plus aucune valeur n'arrivera** ». Après la fermeture :

| Opération | Résultat |
|---|---|
| recevoir alors que le buffer contient encore des valeurs | les valeurs restantes, normalement |
| recevoir sur un channel fermé **et vide** | immédiatement la zéro-valeur, sans bloquer |
| `v, ok := <-ch` | `ok` vaut `false` quand le channel est fermé **et** vide |
| `for v := range ch` | reçoit jusqu'à fermeture **et** vidage, puis sort de la boucle |
| **envoyer** sur un channel fermé | `panic: send on closed channel` |
| fermer un channel **déjà fermé** | `panic: close of closed channel` |
| fermer un channel **nil** | `panic: close of nil channel` |

`range` sur un channel est la forme idiomatique d'un consommateur : la boucle se termine d'elle-même
quand le producteur a fini. Plus besoin de compter les valeurs ni de sentinelle (leçon 2).

Fermer n'est **pas** obligatoire : un channel non fermé est récupéré par le ramasse-miettes quand
plus personne ne le référence. On ferme pour **informer les récepteurs** que c'est fini — donc
seulement quand quelqu'un l'attend (`range`, `ok`).

### Qui ferme ?

Les trois paniques du tableau ont la même cause : une goroutine qui ferme un channel dont elle ne
contrôle pas les envois. D'où la règle :

> **Seul l'émetteur ferme. Jamais le récepteur. Et s'il y a plusieurs émetteurs, aucun d'eux seul :
> un coordinateur ferme quand tous ont fini.**

Le récepteur ne sait pas si un émetteur s'apprête à envoyer ; s'il ferme, cet envoi panique. Avec
plusieurs émetteurs, le premier qui ferme fait paniquer les autres. Le coordinateur classique :

```go
var wg sync.WaitGroup
for _, src := range sources {
	wg.Go(func() { for v := range src { out <- v } })   // N émetteurs
}
go func() {
	wg.Wait()      // quand TOUS ont fini…
	close(out)     // …une seule fermeture, par une goroutine qui n'envoie rien
}()
```

Et dans une fonction productrice, `defer close(out)` en première ligne : le channel est fermé quelle
que soit la sortie de la fonction (y compris un retour anticipé sur erreur).

### Choisir la capacité

| Capacité | Quand |
|---|---|
| **0** (non bufferisé) | par défaut : synchronisation forte, comportement facile à raisonner |
| **1** | un signal qui ne doit pas bloquer l'émetteur (« il y a du nouveau »), un résultat unique rendu par une goroutine qui ne doit pas rester bloquée si personne ne l'attend plus |
| **N connu** | exactement N envois prévus (une goroutine par tâche, chacune envoie un résultat) : aucun ne bloquera |
| **grand** | rarement : masque un problème de débit et retarde sa détection |

Le cas « capacité 1 pour un résultat » est important : il empêche une **fuite de goroutine**.

```go
result := make(chan int, 1)   // si personne ne lit (délai dépassé), l'envoi ne bloque pas
go func() { result <- calcul() }()
```

Avec un channel non bufferisé, si l'appelant abandonne l'attente (délai, leçon 4), la goroutine
reste bloquée **pour toujours** sur son envoi.

## Exemple

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
	"sync"
	"unicode"
)

// lines émet les lignes de text puis FERME out : c'est l'émetteur qui ferme.
func lines(text string, out chan<- string) {
	defer close(out)
	sc := bufio.NewScanner(strings.NewReader(text))
	for sc.Scan() {
		out <- sc.Text()
	}
}

// words découpe chaque ligne en mots ; range s'arrête quand in est fermé et vide.
func words(in <-chan string, out chan<- string) {
	defer close(out)
	for l := range in {
		for _, w := range strings.FieldsFunc(l, func(r rune) bool { return !unicode.IsLetter(r) }) {
			out <- strings.ToLower(w)
		}
	}
}

// merge réunit plusieurs channels en un seul. Plusieurs émetteurs écrivent sur out :
// aucun ne peut le fermer seul. Un coordinateur ferme quand TOUS ont fini.
func merge(ins ...<-chan string) <-chan string {
	out := make(chan string)
	var wg sync.WaitGroup
	for _, in := range ins {
		wg.Go(func() {
			for v := range in {
				out <- v
			}
		})
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}

func main() {
	// 1. Le buffer : les envois ne bloquent pas tant qu'il reste de la place.
	jobs := make(chan int, 3)
	jobs <- 1
	jobs <- 2
	fmt.Println("len", len(jobs), "cap", cap(jobs))
	close(jobs)
	for j := range jobs { // vide le buffer, puis s'arrête : fermé ET vide
		fmt.Print(j, " ")
	}
	v, ok := <-jobs
	fmt.Println("| après fermeture :", v, ok)

	// 2. Deux sources, un pipeline, un consommateur.
	a, b := make(chan string), make(chan string)
	go lines("Le chat dort.\nLe chien aboie !", a)
	go lines("Un chat, deux chats.", b)

	wa, wb := make(chan string), make(chan string)
	go words(a, wa)
	go words(b, wb)

	count := map[string]int{}
	for w := range merge(wa, wb) {
		count[w]++
	}
	fmt.Println(count)
}
```

Sortie :

```
len 2 cap 3
1 2 | après fermeture : 0 false
map[aboie:1 chat:2 chats:1 chien:1 deux:1 dort:1 le:2 un:1]
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `make(chan int, 3)` puis deux envois | Aucun récepteur, et pourtant pas de blocage : le buffer a de la place. Dans la même goroutine, impossible avec un channel non bufferisé. |
| `range jobs` après `close` | Vide le buffer (`1 2`) puis s'arrête. |
| `v, ok := <-jobs` → `0 false` | Fermé et vide : zéro-valeur, `ok` faux, sans bloquer. |
| `defer close(out)` dans `lines` et `words` | L'émetteur ferme, quoi qu'il arrive. La fermeture se **propage** : `lines` ferme `a`, donc la boucle de `words` se termine, donc `words` ferme `wa`… |
| `merge` | Deux émetteurs sur `out` : aucun ne ferme. La goroutine anonyme attend les deux puis ferme. |
| `merge` rend `<-chan string` | L'appelant ne peut que lire : impossible pour lui d'envoyer ou de fermer par erreur. |
| `for w := range merge(…)` | Le consommateur ne sait rien du nombre de sources ni de lignes : il lit jusqu'à la fin. |
| Le `map` dans `main` | Un seul écrivain (`main`), donc pas de verrou. |
| `fmt` trie les clés à l'affichage | La sortie est stable, même si l'ordre d'arrivée des mots varie d'une exécution à l'autre. |

## Erreurs fréquentes

1. **Le récepteur qui ferme.**
2. **Un émetteur parmi plusieurs qui ferme** : les autres paniquent.
3. **Fermer deux fois** (par exemple dans deux chemins d'erreur) : `defer close` en une seule place.
4. **Oublier de fermer** un channel parcouru par `range` : le consommateur attend pour toujours.
5. **Tester `len(ch)`** pour décider d'envoyer.
6. **Un gros buffer** « pour que ça ne bloque pas » : le problème de débit est seulement retardé.
7. **Résultat unique sur un channel non bufferisé** alors que l'appelant peut abandonner : fuite.
8. **Croire qu'un envoi bufferisé signifie « reçu »**.

## Bonnes pratiques Go

- `defer close(out)` en tête de toute fonction productrice.
- Plusieurs émetteurs → un `WaitGroup` et une goroutine de fermeture.
- Les fonctions qui créent un channel rendent `<-chan T`.
- Capacité 0 par défaut, 1 pour un résultat ou un signal, N quand le nombre d'envois est connu.
- `for v := range ch` pour consommer ; `v, ok` quand il faut distinguer la fin dans un `select`.

## Ce que je dois retenir

- Bufferisé : l'envoi bloque si plein, la réception si vide ; plus de garantie de rendez-vous.
- `close(ch)` = « plus rien n'arrivera » ; `range` s'arrête, `ok` devient faux.
- Envoyer sur un fermé, fermer un fermé ou un nil : **panique**.
- **L'émetteur ferme** ; avec plusieurs émetteurs, un coordinateur après `wg.Wait()`.
- Fermer n'est pas obligatoire : seulement pour prévenir les récepteurs.
- Un buffer absorbe des à-coups, pas une lenteur durable.

---

➡️ [Exercices](exercices.md)
