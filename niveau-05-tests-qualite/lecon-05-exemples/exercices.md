# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-05/lecon-05/`.

## Exercices faciles

**E1 — Premier exemple.**
Écrire `ExampleSlug` pour le paquet `slug`. Le faire échouer en modifiant la sortie attendue :
recopier le message. Puis retirer le commentaire `// Output:` : l'exemple est-il encore exécuté ?
Comment le vérifier avec `-v` ?

**E2 — Les noms.**
Pour le paquet `stats` (N3 L2), écrire : un exemple de paquet, un exemple pour `Median`, deux
exemples pour `MinMax` (entiers et chaînes) avec des suffixes. Écrire volontairement
`ExampleMedian_Pair` (majuscule) : que dit `go vet` ? Et `ExampleMediane` ?

**E3 — `go doc -ex`.**
Avec `go doc -ex`, trouver les exemples de `slices.SortFunc`, `errors.As` et `time.Duration.Round`.
Lequel montre un usage que tu ne connaissais pas ? Afficher son programme complet.

**E4 — Sortie non ordonnée.**
Écrire un exemple pour une fonction qui rend une `map[string]int` de fréquences. Essayer avec
`// Output:`, plusieurs fois (`-count=10`) : que se passe-t-il ? Puis `// Unordered output:`.
Puis réécrire l'exemple pour qu'il affiche dans un ordre déterministe : quelle version est la
meilleure **documentation** ?

**E5 — pkg.go.dev en local.**
Lancer `go doc -http` (ou `go run golang.org/x/pkgsite/cmd/pkgsite@latest` dans le module) et
afficher la documentation de ton paquet `queue`. Où apparaissent les exemples ? Que devient le
commentaire placé au-dessus d'une fonction `Example` ?

---

## Exercice intermédiaire — Documenter un paquet pour un inconnu

Prendre le paquet `duration` de la leçon 2 et le rendre **publiable** :

1. Commentaire de paquet dans `doc.go` : à quoi sert le paquet, le format accepté (en
   grammaire simple), ce qui est refusé, et un court bloc de code d'usage.
2. Commentaire sur chaque identifiant exporté, sentinelles comprises : quand chacune est-elle
   rendue ?
3. `Example()` de paquet, `ExampleParse` avec trois appels dont un en erreur (et la manière
   idiomatique de la tester, `errors.Is`), et un exemple suffixé `ExampleParse_composite`.
4. Faire relire `go doc ./duration` et `go doc -all ./duration` par quelqu'un (ou relire
   soi-même le lendemain) : peut-on utiliser le paquet **sans** ouvrir le code ?
5. Ajouter un cas limite à la documentation : que rend `Parse("0h")` ? Le garantir par un exemple.

---

## Défi — L'exemple complet

1. Écrire un fichier `example_retry_test.go` contenant un exemple **complet** : une fonction
   `Retry(attempts int, f func() error) error` est fournie par le paquet ; l'exemple déclare un
   type auxiliaire `flakyService` (qui échoue deux fois puis réussit) et montre l'usage.
2. Vérifier sur la documentation locale que l'exemple est affiché avec ses déclarations.
3. Que faut-il pour qu'un exemple soit considéré comme « complet » par l'outil ? *(Lire
   `go doc testing`, section *Examples*.)*

---

## Quiz

1. Quelle est la signature d'une fonction d'exemple ?
2. Que vérifie `go test` dans un exemple ?
3. Que se passe-t-il si un exemple n'a pas de commentaire `// Output:` ?
4. Comment rattacher un exemple à la méthode `Push` du type `Queue` ?
5. Pourquoi un suffixe d'exemple doit-il commencer par une minuscule ?
6. Quand utiliser `// Unordered output:` ?
7. Pourquoi écrire les exemples dans un paquet `_test` externe ?
8. Que fait `go doc -ex` ?
9. Un exemple peut-il remplacer les tests ?
10. Qu'est-ce qu'un « exemple complet » ?
