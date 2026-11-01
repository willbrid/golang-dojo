# Leçon 12 — Exercices

Code dans `mes-solutions/niveau-06/lecon-12/`.

## Exercices faciles

**E1 — Le `recover` inutile.**
Écrire un `main` avec `defer func() { fmt.Println("récupéré :", recover()) }()`, qui lance une
goroutine qui panique, puis attend 1 s. Qu'affiche le programme ? Le message « récupéré » apparaît-il ?
Quel est le code de sortie (`echo $?`) ?

**E2 — La frontière.**
Écrire `func SafeGo(f func()) <-chan error` qui lance `f` dans une goroutine protégée et rend
l'éventuelle panique sous forme de `*PanicError`. Tester (table) : `f` qui réussit, qui panique avec
une chaîne, avec une erreur (`panic(err)`), avec une valeur nil (`panic(nil)` — que rend `recover`
depuis Go 1.21 ?).

**E3 — `errgroup` et les paniques.**
Montrer qu'une panique dans un `g.Go` d'`errgroup` tue le programme. Écrire un `SafeGroup` qui
enveloppe `errgroup.Group` et convertit les paniques en erreurs avec `safe`. `Wait` doit rendre la
`*PanicError`, et le contexte du groupe doit être annulé comme pour une erreur ordinaire.

**E4 — Ce qui ne se récupère pas.**
Essayer de récupérer : (a) `fatal error: concurrent map writes` (deux goroutines écrivent une map
sans verrou en boucle) ; (b) un débordement de pile (fonction récursive sans fin). Le `recover`
fonctionne-t-il ? Quelle est la seule vraie protection pour chacun ?

**E5 — Le superviseur emballé.**
Retirer le plafond et l'attente de `supervise`, et lui donner un worker qui échoue toujours. Mesurer
le nombre de redémarrages par seconde et l'usage CPU. Remettre l'attente sans le plafond : que se
passe-t-il au bout d'une minute ? Faut-il plafonner l'attente elle-même ?

---

## Exercice intermédiaire — Un pool de workers robuste

Reprendre le worker pool de la leçon 9 et le rendre robuste :

1. Chaque **tâche** est exécutée dans une frontière de récupération : une tâche qui panique produit
   un résultat en erreur (`*PanicError` avec la pile), le worker continue avec la tâche suivante.
2. Un compteur de paniques est exposé (`Panics() int64`, atomique).
3. Si plus de 5 % des tâches paniquent sur les 100 dernières, le pool s'arrête de lui-même et rend
   une erreur (« trop de paniques : bug probable »). Pourquoi est-ce préférable à continuer ?
4. Tests : une tâche qui panique au milieu de 1 000 n'empêche pas les 999 autres ; le taux de paniques
   déclenche l'arrêt ; `-race` et `synctest` passent ; aucune goroutine ne reste après l'arrêt.

---

## Défi — Panique et état partagé

Un cache protégé par un `Mutex` est modifié par une méthode qui panique **entre** deux écritures :

```go
func (c *Cache) Move(from, to string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	v := c.data[from]
	delete(c.data, from)
	c.data[to] = transform(v)      // transform peut paniquer
}
```

1. Avec une frontière de récupération autour de l'appel, le programme continue. Dans quel état est
   le cache ? Le verrou est-il libéré ?
2. Pourquoi est-ce pire qu'un plantage ?
3. Réécrire `Move` pour qu'une panique de `transform` laisse le cache **inchangé** (calculer avant de
   modifier).
4. Énoncer la règle générale : quand un superviseur peut-il relancer un worker sans risque ?

---

## Quiz

1. Que se passe-t-il quand une goroutine panique sans `recover` ?
2. Pourquoi le `recover` de `main` ne protège-t-il pas les autres goroutines ?
3. Où placer le `defer`/`recover` pour protéger une goroutine ?
4. Pourquoi capturer la pile dans le `defer`, et avec quoi ?
5. Pourquoi ne pas mettre de `recover` dans toutes les fonctions ?
6. `net/http` protège-t-il les handlers contre les paniques ? Et `errgroup` ?
7. Citer trois erreurs fatales qu'aucun `recover` ne rattrape.
8. Quels sont les quatre ingrédients d'un superviseur correct ?
9. Quand relancer un worker après une panique est-il dangereux ?
10. Une panique récupérée en production doit-elle déclencher une alerte ?
