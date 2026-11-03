# Module 5 — Exercices

Code dans `mes-solutions/niveau-13/module-05/`. Chaque correction s'accompagne du test qui reproduisait le
défaut **avant** la correction (il doit échouer sur l'ancien code).

## Exercices faciles

**E1 — Corriger l'analyseur, et renforcer la propriété.**
Corriger `Ranges` (une plage décroissante est une erreur). Garder le cas `"10-0"` trouvé par le fuzzing dans
`testdata/fuzz/FuzzRanges/`, et vérifier qu'il est rejoué par `go test`. Renforcer ensuite la propriété du
test de fuzzing : sans erreur, le résultat est trié, sans doublon si les plages ne se chevauchent pas, et
reformater le résultat puis le relire redonne le même résultat. Que trouve le fuzzing en 60 s avec cette
propriété ?

**E2 — La map qui ne rétrécit pas.**
Remplir une `map[int][128]byte` d'un million d'entrées, puis en supprimer toutes sauf dix. Mesurer le tas
vivant après un GC, puis après avoir recopié les dix entrées restantes dans une nouvelle map. Qu'en conclure
pour un cache qui a connu un pic ?

**E3 — L'alias dans une boucle.**

```go
func tagged(items []string, common []string) [][]string {
	var out [][]string
	for _, it := range items {
		out = append(out, append(common, it))
	}
	return out
}
```

Écrire le test qui montre le défaut. Quelle condition sur `common` le déclenche ? Pourquoi n'apparaît-il pas
avec `common := []string{"a", "b"}` mais apparaît avec `common := make([]string, 2, 8)` ? Corriger.

**E4 — L'outil `stress`.**
Installer `golang.org/x/tools/cmd/stress` et l'utiliser sur le test du registre **sans** point d'injection :
`go test -c -o reg.test ./mem/ && stress ./reg.test -test.run TestFlaky`, où `TestFlaky` est un seul
essai qui échoue si les identifiants diffèrent. Combien d'exécutions avant le premier échec ? Comparer avec
`-count` et avec le point d'injection.

**E5 — L'indicateur d'arrêt.**

```go
var stop bool

func worker() {
	for !stop {
		work()
	}
}
```

Une autre goroutine fait `stop = true` au bout d'une seconde. Le travailleur s'arrête-t-il ? Essayer avec un
`work()` vide, puis avec un `work()` qui appelle une fonction non *inlinée*. Lire le code machine
(`go tool objdump`) dans le premier cas. Corriger avec `atomic.Bool`.

---

## Exercice intermédiaire — Le test qui échoue parfois

Ce test « passait » sur le poste de son auteur, qui l'a lancé deux fois. Il échoue pourtant souvent :

```go
type Notifier struct {
	mu   sync.Mutex
	subs []chan string
}

func (n *Notifier) Subscribe() <-chan string {
	ch := make(chan string, 1)
	n.mu.Lock()
	n.subs = append(n.subs, ch)
	n.mu.Unlock()
	return ch
}

func (n *Notifier) Publish(msg string) {
	n.mu.Lock()
	defer n.mu.Unlock()
	for _, ch := range n.subs {
		go func() { ch <- msg }() // ne pas bloquer l'éditeur
	}
}

func TestNotify(t *testing.T) {
	n := &Notifier{}
	a, b := n.Subscribe(), n.Subscribe()
	n.Publish("un")
	n.Publish("deux")
	time.Sleep(10 * time.Millisecond) // laisser le temps aux livraisons
	for _, ch := range []<-chan string{a, b} {
		if got := <-ch; got != "un" {
			t.Errorf("premier message : %q", got)
		}
	}
}
```

1. **Mesurer** la fréquence de l'échec avec `-count=2000` et `-cpu 1`, `4`, `8`. Avec l'une de ces valeurs, il
   échoue **à chaque fois** : laquelle, et pourquoi (indice : dans quel ordre l'ordonnanceur lance-t-il deux
   goroutines qui viennent d'être créées sur le même processeur logique) ?
2. **Comprendre** : pourquoi l'ordre des messages n'est-il pas garanti ? Le `time.Sleep` sert-il à quelque
   chose ? Quel **autre** défaut le test cache-t-il (que deviennent les goroutines qui envoient « deux ») ?
3. **Décider** de la spécification : `Publish` doit-il garantir l'ordre des messages pour un abonné ? Que
   faire d'un abonné lent ?
4. **Corriger** le code et le test. Le test ne doit plus contenir de `time.Sleep`, il utilise `synctest` ou une
   synchronisation explicite, et il passe 100 000 fois de suite.

---

## Défi — La mémoire qui ne correspond à rien

La mémoire de ce service monte pendant une journée, puis se stabilise à un niveau énorme. Pourtant, une
session ne contient que quelques dizaines d'octets utiles :

```go
type Session struct {
	ID    string
	Token []byte // les 32 premiers octets du corps de la requête de connexion
}

var sessions sync.Map // ID → *Session

func login(w http.ResponseWriter, r *http.Request) {
	body, _ := io.ReadAll(r.Body) // un corps de 64 Kio en moyenne (photo de profil encodée)
	s := &Session{ID: rand.Text(), Token: body[:32]}
	sessions.Store(s.ID, s)
	go expire(s.ID) // supprime la session au bout de 24 h
	fmt.Fprintln(w, s.ID)
}

func expire(id string) {
	select {
	case <-time.After(24 * time.Hour):
		sessions.Delete(id)
	}
}
```

1. Écrire une charge qui simule une journée en quelques secondes (durée de vie réduite, connexions
   accélérées). Mesurer le tas et le nombre de goroutines, au fil du temps.
2. Trouver **les** causes par la différence de deux profils de tas (`go tool pprof -base avant.pprof
   apres.pprof`) et par le profil des goroutines. Combien chacune coûte-t-elle **par session** ?
3. Corriger, et mesurer à nouveau.
4. Avec 10 000 connexions par jour, combien de mémoire et de goroutines le service tient-il en régime établi,
   avant et après correction ?

---

## Quiz

1. Pourquoi faut-il rendre un défaut rare fréquent avant de le corriger ?
2. Citer trois façons d'amplifier un défaut de concurrence.
3. Qu'est-ce qu'un point d'injection, et pourquoi le garder dans le code ?
4. Comment le fuzzing aide-t-il à réduire un cas d'échec ?
5. Pourquoi `append(base, x)` peut-il modifier un résultat précédent ?
6. Pourquoi une sous-slice peut-elle retenir beaucoup de mémoire, et que faire ?
7. Pourquoi le compilateur peut-il supprimer une écriture concurrente non synchronisée ?
8. Qu'est-ce qu'une lecture déchirée, et quels types y sont exposés ?
9. Pourquoi `-race` ne trouve-t-il pas le défaut du registre ?
10. Que faire après avoir corrigé un défaut difficile ?
