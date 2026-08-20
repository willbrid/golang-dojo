# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-02/lecon-07/`.

## Exercices faciles

**E1 — Ordre et évaluation.**
Prédire la sortie **avant** d'exécuter :
```go
func main() {
	x := 1
	defer fmt.Println("A", x)
	defer func() { fmt.Println("B", x) }()
	x = 2
	for i := range 3 {
		defer fmt.Println("C", i)
	}
	x = 3
	fmt.Println("fin du corps", x)
}
```
Expliquer chaque ligne de la sortie.

**E2 — `defer` en boucle.**
Écrire un programme qui ouvre 2000 fois le même fichier dans une boucle avec
`defer f.Close()`. Observer ce qui se passe (message `too many open files`). Vérifier la
limite avec `ulimit -n`. Puis corriger de **deux** façons différentes.

**E3 — Résultat nommé.**
Écrire deux versions de la même fonction : l'une avec `defer f.Close()` nu, l'autre avec un
résultat nommé qui récupère l'erreur de `Close`. Provoquer une erreur de fermeture réelle.
*(Indice : écrire dans un fichier ouvert puis fermé une première fois, ou remplir un système
de fichiers en mémoire de petite taille. Si c'est trop difficile à reproduire, simuler avec
un type maison implémentant `io.WriteCloser`.)*

**E4 — `recover` mal placé.**
Écrire trois versions : `recover` appelé directement dans la fonction, `recover` dans une
fonction appelée par la fonction paniquante, `recover` dans un `defer`. Une seule fonctionne.
Vérifier, expliquer précisément pourquoi.

**E5 — `recover` et goroutines.**
Écrire une fonction avec `recover` dans un `defer`, qui lance une goroutine paniquante.
Observer que le programme meurt malgré le `recover`. Relever le message. Corriger.
**Écrire trois phrases sur ce que cela implique pour tout code concurrent.**

---

## Exercice intermédiaire — `resource`

Un gestionnaire de ressources qui garantit la libération, même en cas d'erreur ou de panique.

```go
type Resource struct {
	Name   string
	closed bool
}

func Acquire(name string) (*Resource, error)
func (r *Resource) Close() error       // erreur si déjà fermée
func (r *Resource) Use(op string) error

type Pool struct { /* … */ }
func (p *Pool) WithResource(name string, fn func(*Resource) error) (err error)
```

**Contraintes :**
1. `WithResource` acquiert, appelle `fn`, et libère **toujours** — succès, erreur ou panique.
2. Si `fn` panique, `WithResource` la convertit en `error` **après** avoir libéré la ressource, et l'erreur indique clairement qu'il s'agissait d'une panique.
3. Si `fn` retourne une erreur **et** que `Close` échoue, les deux sont signalées — sans que l'une écrase l'autre. *(Rappel : leçon 6.)*
4. Une double fermeture retourne une erreur et ne panique jamais.
5. Le pool comptabilise les ressources acquises et libérées : à la fin du programme, le compte doit être **exactement** à zéro. L'afficher pour le prouver.
6. Écrire un scénario de démonstration couvrant : cas normal, `fn` en erreur, `fn` qui panique, `Close` en erreur, panique **et** `Close` en erreur.
7. Aucune fuite : après 100 000 appels dont un tiers paniquent, le compteur revient à zéro.

*Ce patron — « exécuter avec une ressource garantie libérée » — est l'équivalent Go du
`with` de Python ou du `try-with-resources` de Java. Go n'en a pas la syntaxe ; il a `defer`
et les fonctions d'ordre supérieur, ce qui donne le même résultat en plus explicite.*

---

## Défi

**a) Convertir des paniques en erreurs.**
Écrire un analyseur d'expressions arithmétiques simple (`3 + 4 * 2`) qui utilise `panic` en
interne pour remonter d'une récursion profonde, et le convertit en `error` à sa frontière
publique. La panique ne doit **jamais** être visible de l'extérieur.
Puis répondre : quel est le risque de cette technique si l'analyseur appelle du code tiers ?
*(Indice : comment distinguer sa propre panique de celle de quelqu'un d'autre ? La réponse
implique un type non exporté.)*

**b) Protection de goroutines.**
Écrire `Go(fn func()) <-chan error` qui lance `fn` dans une goroutine en la protégeant, et
rapporte une éventuelle panique sous forme d'erreur sur le channel, **avec la trace de pile**.
*(Indice : `runtime/debug.Stack()`.)*
Puis expliquer par écrit pourquoi cette fonction est nécessaire, et pourquoi la bibliothèque
standard n'en fournit pas d'équivalent.
*(Les channels arrivent au niveau 6 : se contenter de la structure, quitte à revenir dessus.)*

**c) Coût de `defer`.**
Mesurer le surcoût d'un `defer` : écrire deux fonctions identiques, l'une libérant une
ressource explicitement à chaque `return`, l'autre avec `defer`. Chronométrer dix millions
d'appels.
**Prédire l'écart avant de mesurer.** Puis chercher ce qu'est le « open-coded defer »,
introduit en Go 1.14, et dire dans quels cas il ne s'applique pas — donc dans quels cas le
`defer` reste coûteux. Cela change-t-il la recommandation d'usage ?

---

## Quiz

1. Quand un `defer` s'exécute-t-il exactement ?
2. Dans quel ordre plusieurs `defer` s'exécutent-ils ?
3. Quand les **arguments** d'un appel différé sont-ils évalués ?
4. Un `defer` s'exécute-t-il après une `panic` ? Après `os.Exit` ?
5. Pourquoi `defer` dans une boucle est-il dangereux ? Comment y remédier ?
6. Pourquoi faut-il un résultat **nommé** pour qu'un `defer` modifie l'erreur retournée ?
7. Pourquoi l'erreur de `Close` compte-t-elle en écriture et pas vraiment en lecture ?
8. Où `recover` doit-il être appelé pour avoir un effet ?
9. `recover` protège-t-il les paniques des autres goroutines ?
10. Citer deux usages légitimes de `panic` et deux usages légitimes de `recover`.
11. Quelles erreurs fatales ne sont **pas** rattrapables par `recover` ?
