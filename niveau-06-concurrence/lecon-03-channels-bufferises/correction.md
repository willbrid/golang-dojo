# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Les trois paniques

| Panique | Règle violée |
|---|---|
| `send on closed channel` | quelqu'un d'autre que l'émetteur a fermé, ou un émetteur parmi plusieurs a fermé trop tôt |
| `close of closed channel` | la fermeture n'est pas faite en **un seul** endroit (deux chemins, deux goroutines) |
| `close of nil channel` | le channel n'a jamais été créé avec `make` |

## E2 — Buffer et rendez-vous

Capacité 2 : **3** lignes « envoi » avant la première réception. Les envois 0 et 1 remplissent le
buffer ; la goroutine affiche « envoi 2 » puis bloque sur le troisième envoi. Capacité 0 : **1**
ligne — « envoi 0 », puis blocage jusqu'au rendez-vous. Il faut compter les affichages **avant**
chaque envoi, pas les envois réussis.

## E3 — `ok` et `range`

```go
func drain(ch <-chan int) (sum, n int) {
	for v := range ch {
		sum += v
		n++
	}
	return
}

func drainOK(ch <-chan int) (sum, n int) {
	for {
		v, ok := <-ch
		if !ok {
			return
		}
		sum += v
		n++
	}
}
```

`range` est plus lisible. `v, ok` devient nécessaire dans un **`select`** (leçon 4), qui ne peut
pas contenir de `range` : pour détecter la fermeture d'un des channels qu'il surveille, il faut le
`ok`.

## E4 — Réécrire le pipeline

```go
func read(r io.Reader, out chan<- string, errc chan<- error) {
	defer close(out)
	sc := bufio.NewScanner(r)
	for sc.Scan() {
		out <- sc.Text()
	}
	errc <- sc.Err()       // errc de capacité 1 : jamais bloquant
}

func clean(in <-chan string, out chan<- string) {
	defer close(out)
	for l := range in {
		out <- normalize(l)
	}
}

func count(in <-chan string) map[string]int {
	m := map[string]int{}
	for l := range in {
		for _, w := range strings.Fields(l) {
			m[w]++
		}
	}
	return m
}
```

Disparaissent : le type `line`, tous les tests `if l.eof`, la propagation manuelle de la fin. Le
dernier étage n'a même plus besoin d'être une goroutine. L'erreur de lecture a son **propre**
channel, de capacité 1, lu par `main` après `count` : séparer données et erreur est plus clair que
de les mélanger. La leçon 9 (`errgroup`) donnera une forme plus générale.

## E5 — La fuite du résultat unique

Mesuré typiquement : `NumGoroutine()` ≈ **1 001** après 500 ms avec le channel non bufferisé —
chaque goroutine a fini son calcul mais reste bloquée sur son envoi, puisque personne ne lit. Avec
`make(chan int, 1)` : **1** (seule `main` reste) ; chaque goroutine dépose sa valeur dans le buffer
et se termine, et le ramasse-miettes récupère le channel quand plus personne ne le référence.

C'est la forme la plus courante de fuite de goroutine (leçon 10) : une goroutine qui rend un
résultat que plus personne n'attend.

## Exercice intermédiaire — `merge` et `split`

```go
func Merge[T any](ins ...<-chan T) <-chan T {
	out := make(chan T)
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

func Split[T any](in <-chan T, n int) []<-chan T {
	outs := make([]chan T, n)
	ros := make([]<-chan T, n)     // vues en lecture seule pour l'appelant
	for i := range outs {
		outs[i] = make(chan T)
		ros[i] = outs[i]
	}
	go func() {
		defer func() {
			for _, o := range outs {
				close(o)
			}
		}()
		i := 0
		for v := range in {
			outs[i] <- v
			i = (i + 1) % n
		}
	}()
	return ros
}
```

- `Merge()` sans entrée : `wg.Wait()` rend immédiatement, la sortie est fermée — correct.
- **Attention à `Split`** : il distribue en tourniquet **sur des channels non bufferisés**. Si
  l'appelant lisait les sorties **l'une après l'autre** (toute la première, puis la deuxième…),
  `Split` resterait bloqué sur la deuxième sortie pendant qu'on vide la première : interblocage.
  `Merge(Split(…)...)` fonctionne parce que `Merge` lit toutes les sorties **en même temps**. Cette
  contrainte doit être **documentée**.
- Test borné dans le temps : lire le résultat dans une goroutine et attendre avec un délai :

  ```go
  done := make(chan []int, 1)
  go func() { done <- slices.Collect(...) }()   // ou une boucle range
  select {
  case got := <-done:
  	…
  case <-time.After(time.Second):
  	t.Fatal("blocage : une sortie n'a pas été fermée")
  }
  ```

  ou lancer `go test -timeout 5s`, qui fait échouer le binaire de test entier avec la pile de
  **toutes** les goroutines — précieux pour voir laquelle attend quoi.

## Défi — La bonne capacité

Résultats qualitatifs attendus :

- Capacité 0 ou 1 : l'émetteur attend pendant chaque rafale (jusqu'à ~45 ms), mais le temps total
  reste dicté par le consommateur.
- Capacité **50** (la taille d'une rafale) : l'émetteur n'attend presque plus ; le consommateur
  rattrape pendant les 50 ms de silence (50 × 0,9 = 45 ms < 50 ms).
- Capacité 1 000 : **aucun** gain supplémentaire — le buffer n'est jamais plus rempli que par une
  rafale. Elle ne fait que réserver de la mémoire et, surtout, masquerait un problème si le débit
  changeait.

À 1,1 ms par valeur, le consommateur est **plus lent** en moyenne que le producteur (1 ms) : le
buffer se remplit inexorablement, **quelle que soit** sa taille, puis l'émetteur bloque à chaque
envoi. Un buffer ne compense que des variations ; si le débit moyen d'entrée dépasse le débit de
sortie, il faut ralentir l'entrée (*backpressure*), rejeter, ou ajouter des consommateurs. C'est le
sujet de la leçon 11.

## Quiz

1. Quand le buffer est plein.
2. La garantie que le récepteur a reçu la valeur quand l'envoi se termine.
3. Les valeurs restantes, normalement, avec `ok` vrai.
4. Quand le channel est fermé **et** vide.
5. Envoyer sur un channel fermé, fermer un channel fermé, fermer un channel nil.
6. L'émetteur ; avec plusieurs émetteurs, une goroutine coordinatrice qui ferme après qu'ils ont
   tous fini (`wg.Wait()`).
7. Non : seulement pour informer des récepteurs qui attendent la fin.
8. La valeur est périmée aussitôt lue : une autre goroutine peut remplir le buffer entre le test et
   l'envoi.
9. Pour que la goroutine puisse déposer sa valeur et se terminer même si plus personne ne l'attend.
10. Non : il absorbe les à-coups ; le débit moyen reste celui du maillon le plus lent.
