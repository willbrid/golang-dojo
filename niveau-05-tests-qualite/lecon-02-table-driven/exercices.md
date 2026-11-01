# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-05/lecon-02/`.

## Exercices faciles

**E1 — Convertir.**
Réécrire `TestSlug` (leçon 1, E1) sous forme de table. Ajouter **dix** cas, dont au moins six cas
limites (vide, uniquement de la ponctuation, majuscules accentuées, chiffres, tirets multiples,
emoji…). Combien de bugs de `Slug` trouves-tu ?

**E2 — Noms en double.**
Mettre deux cas avec le même `name`. Lancer `go test -v`. Comment le second est-il nommé ?
Pourquoi est-ce un problème pour `-run` ?

**E3 — `reflect.DeepEqual`.**
Pour une fonction qui rend un `[]string`, comparer `nil` et `[]string{}` avec `reflect.DeepEqual`,
puis avec `slices.Equal`. Quel outil considère qu'ils sont égaux ? Lequel choisir quand le
contrat d'une API dit « rend une liste vide, jamais nil » ? *(Lien avec N4 L6 : `null` contre `[]`.)*

**E4 — La variable de boucle.**
Écrire un test table avec `t.Parallel()` qui affiche `tt.name` dans chaque sous-test. Lancer avec
`go 1.27` dans `go.mod`, puis avec `go 1.21`. Comparer les sorties. Que se passerait-il pour les
**vérifications** dans le second cas ?

**E5 — Table d'erreurs.**
Pour `ParseDue` du projet 2 (ou une fonction équivalente de ton choix), écrire une table avec un
champ `wantErr error`. Au moins quatre sentinelles distinctes doivent être testées.

---

## Exercice intermédiaire — Spécifier par la table

On veut une fonction `SemverCompare(a, b string) (int, error)` qui compare deux versions
sémantiques (`v1.2.3`, `1.10.0-rc.1`, `2.0.0+build5`).

**Démarche imposée — d'abord la table, ensuite le code :**
1. Écrire **uniquement** la table de tests, avec au moins 20 cas : égalités, chaque composante
   qui départage, nombres à plusieurs chiffres (`1.10` > `1.9`), préfixe `v` facultatif,
   pré-versions (`1.0.0-alpha` < `1.0.0`), comparaison des identifiants de pré-version
   (numériques contre alphanumériques), métadonnées de build ignorées (`+…`), entrées invalides.
2. Lancer les tests sur une implémentation qui fait `return 0, nil` : combien échouent ?
3. Écrire le code jusqu'à ce que tout passe. **Ne pas modifier la table** pendant cette phase,
   sauf pour corriger une erreur de spécification — et le noter.
4. Lire la spécification officielle (semver.org, section 11) et vérifier que la table la couvre.
   Quels cas manquaient ?
5. Ajouter une propriété : `SemverCompare(a, b) == -SemverCompare(b, a)` pour toutes les paires
   de versions valides de la table. L'écrire comme un second test qui **réutilise** la table.

---

## Défi — Quand la table devient illisible

Voici le début d'une table trouvée dans un projet :

```go
tests := []struct {
	name       string
	input      string
	setup      func(*Store)
	check      func(*testing.T, *Store, Result)
	wantErr    bool
	skipOnCI   bool
	parallel   bool
}{ … }
```

1. Qu'est-ce qui a mal tourné ? Identifier précisément les champs qui signalent que la table
   n'est plus le bon outil.
2. Proposer une réorganisation : quels cas restent dans une table, lesquels deviennent des tests
   séparés ?
3. Énoncer une règle en une phrase pour décider entre table et tests séparés.

---

## Quiz

1. Quels sont les éléments du patron table-driven ?
2. Pourquoi une slice de cas plutôt qu'une map ?
3. Pourquoi des champs nommés dans les littéraux de cas ?
4. Comment tester avec la même table les cas avec et sans erreur ?
5. Pourquoi ne pas comparer le texte d'une erreur ?
6. Quel piège de `reflect.DeepEqual` concerne les slices ?
7. Que fait `t.Parallel()` dans un sous-test ?
8. Pourquoi écrivait-on `tt := tt` avant Go 1.22, et pourquoi ne plus le faire ?
9. Quand ne pas utiliser de table ?
10. Que faire de la table quand on corrige un bug ?
