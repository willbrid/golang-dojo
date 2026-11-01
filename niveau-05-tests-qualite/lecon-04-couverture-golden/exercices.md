# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-05/lecon-04/`.

## Exercices faciles

**E1 — Lire la vue HTML.**
Lancer `go test -coverprofile=cover.out` puis `go tool cover -html=cover.out` sur le paquet
`duration` de la leçon 2. Quelles lignes sont rouges ? Pour chacune : ajouter un cas à la table,
ou écrire en une phrase pourquoi elle peut rester rouge.

**E2 — 100 % et faux.**
Écrire une fonction `Average(xs []float64) float64` **boguée** (division par zéro sur une liste
vide, ou mauvais calcul sur un seul élément) et un test qui atteint **100 %** de couverture sans
détecter le bug. Puis ajouter le cas qui le révèle. Le pourcentage a-t-il changé ?

**E3 — Couverture par condition.**
Pour `func ok(a, b bool) bool { if a && b { return true }; return false }`, combien de cas faut-il
pour 100 % de couverture **de lignes** ? Combien pour tester toutes les combinaisons ? Que conclure
sur ce que mesure `-cover` ?

**E4 — Golden file instable.**
Modifier `Render` pour qu'il ajoute en en-tête `Généré le <date du jour>`. Lancer le test deux
jours de suite (ou simuler en changeant l'heure système… ou mieux, en raisonnant). Corriger
**sans** retirer l'en-tête.

**E5 — `-update` aveugle.**
Introduire un bug dans `Render` (inverser deux colonnes). Lancer `go test -update`. Le test
passe-t-il ensuite ? Qu'est-ce qui, dans le flux de travail, aurait dû arrêter le bug ?

---

## Exercice intermédiaire — Golden files pour `todo list`

Pour le projet 2, écrire les tests de l'affichage de `todo list` (et de `--json`) avec des golden
files.

**Contraintes :**
1. Au moins 6 cas : liste vide, une tâche, tâches en retard / aujourd'hui / demain / sans
   échéance, tâches terminées avec et sans `--all`, filtres combinés, titre très long.
2. L'horloge et le fuseau sont fixés ; les golden files sont identiques sur toutes les machines,
   quel que soit le fuseau du système (vérifier avec `TZ=America/New_York go test ./...`).
3. Drapeau `-update` ; message d'échec qui affiche un diff lisible (écrire une petite fonction de
   diff ligne à ligne, ou utiliser `go-cmp`).
4. Les entrées sont aussi dans `testdata/` (fichiers JSON de tâches), pas dans le code.
5. Mesurer la couverture du paquet `cli` avant et après : quelles fonctions restent sous 50 % ?

---

## Défi — Couverture d'intégration

Depuis Go 1.20, un **binaire** peut être instrumenté pour mesurer la couverture d'exécutions
réelles, hors `go test`.

1. Lire `go help testflag` et la documentation de `go build -cover` (chercher `GOCOVERDIR`).
2. Construire le binaire de `wordstat` avec `-cover`, l'exécuter cinq fois avec des arguments
   différents (dont des erreurs) en définissant `GOCOVERDIR`.
3. Produire un rapport avec `go tool covdata percent` puis convertir en profil texte
   (`go tool covdata textfmt`) pour l'ouvrir avec `go tool cover -html`.
4. Comparer avec la couverture des tests unitaires. Quelles lignes ne sont couvertes que par
   l'une des deux méthodes ? Qu'est-ce que ça dit des tests unitaires ?

---

## Quiz

1. Quelle commande ouvre la couverture ligne par ligne dans un navigateur ?
2. Une ligne « verte » est-elle une ligne vérifiée ?
3. La couverture mesure-t-elle les conditions d'un `if a && b` ?
4. Pourquoi viser 100 % de couverture est-il souvent contre-productif ?
5. Que fait `-coverpkg=./...` ?
6. Quelle particularité a le répertoire `testdata` ?
7. Dans quel répertoire courant les tests s'exécutent-ils ?
8. Qu'est-ce qu'un golden file ? Quand l'utiliser ?
9. Quelle est l'étape indispensable après un `-update` ?
10. Citer trois sources de non-déterminisme qui cassent un golden file.
