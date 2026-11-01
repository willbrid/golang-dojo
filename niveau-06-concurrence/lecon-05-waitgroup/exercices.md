# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-06/lecon-05/`. Tout avec `-race` et `go vet`.

## Exercices faciles

**E1 — Les quatre pièges.**
Écrire quatre petits programmes, un par piège du cours. Pour chacun : que dit `go vet` ? que dit
`go run -race` ? que se passe-t-il à l'exécution ? Présenter en tableau. Lequel aucun outil ne
détecte-t-il ?

**E2 — Moderniser.**
Écrire une fonction avec la forme `Add`/`go`/`defer Done`, puis lancer `go fix -diff ./...`. Que
propose le modernisateur `waitgroupgo` ? Dans quel cas **ne** peut-il **pas** convertir ?
*(Essayer avec un `Add(len(items))` avant la boucle.)*

**E3 — `append` concurrent.**
Collecter les résultats de 1 000 goroutines avec `results = append(results, r)` sur un slice
partagé. Combien de résultats obtient-on (5 essais) ? Que dit `-race` ? Corriger de deux façons :
case par goroutine, et channel bufferisé.

**E4 — Attendre avec un délai.**
`wg.Wait()` n'a pas de délai. Écrire `func WaitTimeout(wg *sync.WaitGroup, d time.Duration) bool`
qui rend `false` si le groupe n'a pas fini à temps. *(Indice : `Wait` dans une goroutine qui ferme
un channel, et `select`.)* Quel est le coût caché de cette fonction quand le délai expire ?

**E5 — Groupes imbriqués.**
Un répertoire contient des sous-répertoires, chacun des fichiers. Lancer une goroutine par
sous-répertoire, qui elle-même lance une goroutine par fichier (pour compter les octets). Un seul
`WaitGroup` suffit-il ? Écrire les deux versions (un seul groupe, deux niveaux de groupes) et
comparer leur lisibilité.

---

## Exercice intermédiaire — Vérificateur de liens en parallèle

Reprendre `mdlinks` (N4 L2, exercice intermédiaire) et vérifier les liens relatifs de **tous** les
fichiers Markdown d'une arborescence, en parallèle.

**Contraintes :**
1. Une goroutine par fichier Markdown, avec `wg.Go`.
2. Les résultats sont affichés **triés** par fichier puis par ligne, quel que soit l'ordre de fin.
3. Les erreurs de lecture d'un fichier n'arrêtent pas les autres ; elles sont réunies avec
   `errors.Join` et affichées à la fin.
4. `-race` silencieux sur le dépôt `golang-dojo` lui-même.
5. Mesurer le temps contre la version séquentielle. Le gain est-il à la hauteur du nombre de
   fichiers ? Pourquoi ?
6. Que faudrait-il pour limiter à 8 le nombre de fichiers traités simultanément ? Écrire une
   première version avec un channel bufferisé de capacité 8 utilisé comme **sémaphore** (prendre
   un jeton avant le travail, le rendre après). La leçon 9 généralisera.

---

## Défi — Implémenter un `WaitGroup`

Écrire son propre `type Group struct { … }` avec `Add`, `Done`, `Wait`, **uniquement avec des
channels** (pas de `sync`).

1. Une première version avec une goroutine propriétaire du compteur (patron de la leçon 2, E5).
2. `Wait` doit pouvoir être appelé par **plusieurs** goroutines à la fois, et toutes doivent être
   réveillées. *(Indice : fermer un channel.)*
3. Un `Done` de trop doit paniquer avec un message clair.
4. Comparer les performances avec `sync.WaitGroup` sur 100 000 `Add`/`Done` (benchmark, N5 L6).
   Pourquoi la version de la bibliothèque standard est-elle beaucoup plus rapide ? *(Elle repose sur
   des opérations atomiques — leçon 7.)*

---

## Quiz

1. Que fait chacune des méthodes `Add`, `Done`, `Wait` et `Go` ?
2. Quelle garantie mémoire `Wait` apporte-t-il ?
3. Pourquoi `Add` doit-il être appelé avant le `go` ?
4. Pourquoi `Done` en `defer` ?
5. Que se passe-t-il avec un `Done` de trop ?
6. Pourquoi ne jamais copier un `WaitGroup` ? Quel outil le détecte ?
7. `WaitGroup` limite-t-il le nombre de goroutines simultanées ?
8. Comment collecter les résultats de N goroutines sans verrou ?
9. Comment attendre un `WaitGroup` avec un délai ?
10. Que fait `go fix` des anciennes formes `Add`/`Done` ?
