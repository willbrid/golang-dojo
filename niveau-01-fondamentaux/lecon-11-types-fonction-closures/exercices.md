# Leçon 11 — Exercices

Code dans `mes-solutions/niveau-01/lecon-11/`.

## Exercices faciles

**E1 — Fonction en variable.**
Déclarer `var op func(int, int) int`, l'appeler **avant** de l'affecter (constater la
panique, relever le message), puis l'affecter successivement à trois opérations et
l'appeler. Écrire ensuite une garde qui évite la panique.

**E2 — Ordre supérieur.**
Écrire `Apply(nums []int, f func(int) int) []int` et `Filter(nums []int, keep func(int) bool) []int`.
Les utiliser avec une fonction nommée puis avec une fonction anonyme.
**Contrainte : l'entrée ne doit jamais être modifiée.** Le vérifier explicitement.

**E3 — Compteur.**
Écrire `Counter() func() int` et vérifier que deux compteurs sont indépendants. Puis écrire
`CounterFrom(start, step int) func() int` — une closure qui capture deux paramètres.
Où vit la variable capturée ? Expliquer en une phrase.

**E4 — Le piège de la capture.**
Écrire les deux versions ci-dessous, prédire leur sortie, vérifier, expliquer l'écart :
```go
// A
for i := range 3 { fns = append(fns, func() { fmt.Print(i) }) }
// B
x := 0
for i := range 3 { x = i; fns = append(fns, func() { fmt.Print(x) }) }
```

**E5 — Map de fonctions.**
Remplacer ce `switch` par une `map[string]func(float64, float64) float64` :
```go
switch op {
case "+": return a + b
case "-": return a - b
case "*": return a * b
case "/": return a / b
}
```
Gérer l'opérateur inconnu et la division par zéro. Quels avantages et quels inconvénients
par rapport au `switch` ? Répondre en trois lignes.

---

## Exercice intermédiaire — `retry`

Écrire une fonction de réessai — un usage réel et fréquent des types fonction.

```go
// Retry exécute op jusqu'à attempts fois, en attendant delay entre deux essais.
func Retry(attempts int, delay time.Duration, op func() error) error
```

**Contraintes :**
1. `attempts <= 0` est une erreur de programmation : retourner une erreur explicite, ne jamais paniquer.
2. Aucune attente **après** la dernière tentative : `Retry(3, time.Second, …)` qui échoue doit durer 2 s, pas 3 s.
3. L'erreur finale indique le nombre de tentatives et **enveloppe** la dernière erreur avec `%w` (leçon 10).
4. Ajouter `RetryBackoff` où le délai **double** à chaque échec.
5. Dans `main`, démontrer avec une opération qui échoue deux fois puis réussit — construite avec une **closure sur un compteur**. C'est le point de l'exercice.
6. `Retry` ne doit **rien afficher**. Si l'on veut observer les tentatives, comment faire sans que la bibliothèque impose son format de logs ? **Trancher, implémenter, justifier en commentaire.**

*Ce patron est réellement utilisé en production. Il reviendra au niveau 6 avec `context`
(pour être annulable) et au niveau 10 avec le jitter et les circuit breakers.*

---

## Défi

**a) `Compose` et `ComposeErr`.**
```go
type Transform func(string) string
func Compose(ts ...Transform) Transform
func ComposeErr(ts ...func(string) (string, error)) func(string) (string, error)
```
`Compose()` sans argument doit retourner une transformation **valide** (l'identité), pas
`nil` — sans cas particulier dans le code, si possible. `ComposeErr` s'arrête à la première
erreur en indiquant **quelle étape** a échoué.

**b) Aliasing de closure.**
`Compose` construit-il sa chaîne au moment de l'appel de `Compose`, ou au moment de l'appel
du résultat ? Les deux sont implémentables. Écrire les deux, puis démontrer par un programme
la différence de comportement quand le slice `ts` est modifié entre les deux appels.
Laquelle est le bon contrat pour une valeur qu'on distribue à d'autres ? Pourquoi ?

**c) Options fonctionnelles.**
Implémenter le patron du cours pour un type `Server` avec cinq réglages
(`port`, `host`, `timeout`, `maxConns`, `tls bool`). Puis répondre par écrit :
comment une option pourrait-elle **échouer** (port hors bornes, par exemple) ?
Comparer trois conceptions : `Option func(*Server)`, `Option func(*Server) error`, et une
validation groupée à la fin du constructeur. Laquelle retenir ?
*(Chercher ensuite comment `grpc.NewServer` s'y prend réellement.)*

---

## Quiz

1. Quel est le type de `func add(a, b int) int` vu comme valeur ?
2. Que se passe-t-il si on appelle une variable de fonction valant `nil` ?
3. Une closure capture-t-elle une copie de la variable, ou la variable ?
4. Où vit une variable locale capturée par une closure retournée ? Qui décide ?
5. Pourquoi le correctif de Go 1.22 ne règle-t-il pas le cas B de l'exercice E4 ?
6. Citer trois avantages d'un type fonction nommé.
7. Comment un type fonction peut-il satisfaire une interface ? Quel exemple célèbre dans la stdlib ?
8. Qu'est-ce qu'un décorateur, et à quel mécanisme du niveau 7 correspond-il ?
9. Pourquoi le patron des options fonctionnelles existe-t-il en Go ?
10. Une closure alloue-t-elle ? Est-ce une raison de l'éviter ?
