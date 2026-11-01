# Leçon 9 — Exercices

Code dans `mes-solutions/niveau-06/lecon-09/`. `go get golang.org/x/sync` dans le module. Tout avec
`-race`.

## Exercices faciles

**E1 — La fuite du pipeline.**
Retirer le `case <-ctx.Done()` de `generate` dans l'exemple. Faire consommer seulement les 2
premiers résultats par `main`, puis `cancel()`. Compter `runtime.NumGoroutine()` après une pause.
Remettre le `case` et recompter. Écrire le même constat sous forme de test avec `testing/synctest`.

**E2 — Réécrire la leçon 8.**
Réécrire `Search` (leçon 8, exercice intermédiaire) avec `errgroup.WithContext`. Combien de lignes
disparaissent ? Les tests `synctest` passent-ils encore sans modification ?

**E3 — `SetLimit`.**
Lancer 100 tâches qui durent 10 ms chacune dans un `errgroup` avec `SetLimit(10)`. Mesurer la durée
totale (ou la vérifier avec `synctest`). Ajouter un compteur atomique du nombre de tâches **actives**
et vérifier que son maximum ne dépasse jamais 10.

**E4 — L'ordre.**
Dans un worker pool à 4 workers, envoyer 20 tâches de durées aléatoires. Afficher l'ordre d'arrivée
des résultats. Puis garantir l'ordre d'entrée de **deux** façons : en triant à la fin, et en écrivant
dans un slice indexé. Laquelle permet d'afficher les résultats **au fur et à mesure**, sans attendre
la fin ? *(Indice : aucune des deux — chercher une troisième.)*

**E5 — Les erreurs perdues.**
Dans un `errgroup`, faire échouer 3 tâches sur 10. Que rend `Wait` ? Écrire une variante qui rend
**toutes** les erreurs avec `errors.Join`, tout en annulant les autres tâches à la première.

---

## Exercice intermédiaire — Indexeur de fichiers concurrent

Écrire `index DIR` qui calcule, pour chaque fichier texte d'une arborescence, son nombre de lignes,
de mots, et son empreinte SHA-256, puis affiche un tableau trié par chemin et un total.

**Architecture imposée — un pipeline :**
1. **Parcours** (`filepath.WalkDir`) → channel de chemins.
2. **Worker pool** de N workers (option `-workers`, défaut `runtime.GOMAXPROCS(0)`) : lecture en
   flux (`io.Copy` dans un `MultiWriter` compteur + hash, N4 L5) → channel de résultats.
3. **Collecte** : un seul consommateur, qui range et trie.

**Contraintes :**
1. Un fichier illisible produit une erreur dans le résultat, n'arrête rien.
2. Ctrl-C (`signal.NotifyContext(ctx, os.Interrupt)`, à découvrir dans `go doc os/signal`) annule le
   contexte : le programme s'arrête **proprement** en moins de 100 ms, en affichant le total partiel,
   sans goroutine bloquée.
3. Mémoire bornée quelle que soit la taille des fichiers.
4. Mesurer la durée sur un gros répertoire (par exemple `$(go env GOROOT)/src`) pour 1, 2, 4, 8, 16
   workers. Tracer. Où est le plafond, et pourquoi ?
5. Tests : un `fstest.MapFS` (N4 L7) pour le parcours ; un test d'annulation avec `synctest`.

---

## Défi — Pipeline par lots

Reprendre le pipeline de comptage de mots de la leçon 3 (étages `lines` → `words` → comptage).

1. Le benchmarker sur 50 Mo de texte contre une boucle séquentielle. Constater qu'il est plus lent.
2. Le réécrire pour que les étages échangent des **lots** de 1 000 lignes (`[]string`) au lieu de
   lignes individuelles. Mesurer pour des lots de 10, 100, 1 000, 10 000.
3. Paralléliser l'étage `words` avec un worker pool de 4. Le comptage final fusionne des maps
   partielles (une par worker), sans verrou.
4. Le pipeline bat-il maintenant la boucle séquentielle ? De combien ? Expliquer le rôle respectif de
   la taille des lots et du nombre de workers.

---

## Quiz

1. Quelles sont les trois règles d'un étage de pipeline ?
2. Que se passe-t-il si un étage n'écoute pas l'annulation et que l'aval s'arrête ?
3. Qu'est-ce que le fan-out ? le fan-in ?
4. Pourquoi un worker pool plutôt qu'une goroutine par tâche ?
5. L'ordre des résultats d'un worker pool est-il garanti ?
6. Que fait `errgroup.WithContext` de plus qu'un `WaitGroup` ?
7. Que fait `SetLimit` ? Et `TryGo` ?
8. Que rend `g.Wait()` quand plusieurs tâches échouent ?
9. Pourquoi les tâches d'un `errgroup` doivent-elles respecter `ctx` ?
10. Quand ne faut-il utiliser aucun de ces patrons ?
