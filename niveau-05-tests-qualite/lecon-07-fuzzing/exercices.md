# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-05/lecon-07/`. Pour chaque séance de fuzzing, noter la durée,
l'entrée trouvée, et ce qu'elle révèle.

## Exercices faciles

**E1 — Rejouer la séance.**
Retaper l'exemple du cours **dans sa première version** (format `<nombre><caractère>`, sans limite)
et lancer `go test -run '^$' -fuzz FuzzRoundTrip`. Combien de temps le fuzzer met-il à trouver le
bug n° 1 ? Est-ce la même entrée que dans le cours ? Après correction, retrouver les bugs 2 et 3.

**E2 — Invariant de `Slug`.**
Écrire `FuzzSlug` qui vérifie, pour toute entrée : (a) pas de panique ; (b) la sortie ne contient
que `[a-z0-9-]` ; (c) pas de `-` en tête, en fin, ni doublé ; (d) `Slug(Slug(x)) == Slug(x)`
(idempotence). Lancer 60 secondes. Qu'as-tu trouvé ?

**E3 — Le corpus.**
Après une séance qui a trouvé un bug, lancer `go test -v ./…` (sans `-fuzz`). Où apparaît l'entrée
enregistrée ? Supprimer le fichier de `testdata/fuzz/` : que perd-on ?

**E4 — Équivalence.**
Écrire une version « optimisée » de `strings.Fields` (par exemple sans allocation intermédiaire)
et une cible qui vérifie l'**équivalence** avec `strings.Fields` sur toute entrée. Le fuzzer
trouve-t-il une différence ? *(Indice : quels caractères Unicode sont des espaces ?)*

**E5 — Non déterminisme.**
Écrire une cible qui appelle `rand.IntN` dans son corps et échoue une fois sur mille. Le fuzzer
trouve-t-il une entrée ? Peut-on la rejouer avec la commande affichée ? Conclusion.

---

## Exercice intermédiaire — Fuzzer `duration.Parse`

Reprendre le paquet `duration` de la leçon 2 (table de onze cas, tous verts).

1. Écrire `FuzzParse` avec la propriété : *si `Parse` ne rend pas d'erreur, la durée n'est jamais
   négative*. Corpus de départ : les entrées valides de la table.
2. Lancer 60 secondes. Recopier l'entrée trouvée et la valeur rendue. Expliquer le mécanisme.
3. Corriger `Parse` avec une **nouvelle sentinelle**, sans calcul en flottant. Attention : le test
   de débordement lui-même ne doit pas déborder.
4. Ajouter l'entrée trouvée à la **table** de la leçon 2 (en plus de `testdata/fuzz`). Pourquoi
   les deux ?
5. Ajouter une seconde propriété : pour toute durée `d` entre 0 et 1 000 heures, arrondie à la
   seconde, `Parse(Format(d)) == d`, où `Format` est une fonction à écrire qui produit le format
   du paquet (`"1h30min"`). Quelle cible écrire, avec quel type d'argument ?

---

## Défi — Fuzzer un décodeur de la bibliothèque standard

1. Écrire une cible qui vérifie l'aller-retour `json.Marshal` → `json.Unmarshal` pour une struct
   `{Name string; Age int; Tags []string}` construite à partir des arguments du fuzzer
   (`f.Fuzz(func(t *testing.T, name string, age int, tag string)`).
2. Lancer 2 minutes. Trouve-t-il une différence ? Si oui, est-ce un bug de `encoding/json` ou une
   propriété mal formulée ? *(Penser à l'UTF-8 invalide dans `name`, et relire N4 L6.)*
3. Écrire la même cible avec `encoding/json/v2`. Le comportement sur l'UTF-8 invalide est-il le
   même ? Lire la documentation de `jsontext` sur ce point.

---

## Quiz

1. Quelle est la signature d'une cible de fuzzing ?
2. Que fait `go test` sans `-fuzz` sur une cible de fuzzing ?
3. Pourquoi le fuzzing de Go est-il dit « guidé par la couverture » ?
4. Citer quatre propriétés qu'une cible peut vérifier.
5. Que se passe-t-il quand le fuzzer trouve une entrée fautive ?
6. Pourquoi commiter `testdata/fuzz/` ?
7. Que signifie un échec « fuzzing process hung or terminated unexpectedly » ?
8. Quel type de code fuzzer en priorité ?
9. Pourquoi une cible doit-elle être déterministe ?
10. Qu'est-ce qu'une bombe de décompression, et comment s'en protéger ?
