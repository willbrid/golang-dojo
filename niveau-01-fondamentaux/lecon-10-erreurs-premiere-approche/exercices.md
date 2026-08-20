# Leçon 10 — Exercices

Code dans `mes-solutions/niveau-01/lecon-08/`.

## Exercices faciles

**E1 — Signature.**
Écrire `Divide(a, b float64) (float64, error)` qui refuse la division par zéro.
Puis répondre par écrit : que doit valoir le premier résultat quand l'erreur est non nulle,
et pourquoi ce choix plutôt que `NaN` ?

**E2 — Contexte.**
Écrire `ReadFirstLine(path string) (string, error)` qui lit la première ligne d'un fichier.
Enrichir chaque erreur du chemin concerné avec `%w`. Tester sur : un fichier existant, un
fichier absent, un répertoire, un fichier vide.
**Recopier les quatre messages d'erreur obtenus** et vérifier qu'ils sont tous compréhensibles
sans lire le code.

**E3 — Critique de messages.**
Corriger ces cinq messages selon les conventions Go, et expliquer chaque correction :
```go
errors.New("Erreur: Impossible d'ouvrir le fichier.")
fmt.Errorf("échec")
errors.New("ERREUR FATALE !!!")
fmt.Errorf("erreur lors de la lecture: %v", err)
errors.New("Le user n'existe pas")
```

**E4 — Sentinelle.**
Créer `ErrNotFound`, une fonction `FindUser(id int) (string, error)` qui la retourne pour
les identifiants inconnus, et un appelant qui la distingue des autres erreurs avec
`errors.Is`. Puis envelopper l'erreur d'un niveau supplémentaire et **vérifier qu'`errors.Is`
fonctionne toujours** — c'est le point de l'exercice.

**E5 — `%w` contre `%v`.**
Reprendre E4, remplacer `%w` par `%v` dans l'enveloppement, et constater ce que
`errors.Is` retourne. Expliquer en deux phrases.

---

## Exercice intermédiaire — `csvcheck`

Valider un fichier CSV simple et rapporter **toutes** les erreurs, pas seulement la première.

```
$ go run . donnees.csv
ligne 3 : 2 champs attendus, 3 trouvés
ligne 5 : champ « age » : strconv.Atoi: parsing "abc": invalid syntax
ligne 8 : champ « name » vide
3 erreurs sur 12 lignes
```

**Contraintes :**
1. Format attendu : `name,age`, une en-tête obligatoire, l'âge entre 0 et 150.
2. **Toutes** les erreurs sont collectées et rapportées ; le traitement ne s'arrête pas à la première. *(Quelle structure de données pour ça ? Quel type de retour ? C'est la vraie question de conception de l'exercice.)*
3. Les erreurs de format vont sur `os.Stderr`, le résumé aussi. Code de sortie 1 s'il y a au moins une erreur.
4. Une erreur d'ouverture de fichier est fatale et s'arrête immédiatement — la distinction entre « erreur fatale » et « erreur collectée » doit apparaître clairement dans le code.
5. Chaque message d'erreur indique le numéro de ligne et le champ concerné.
6. Aucun `panic`.
7. La fonction de validation ne fait **aucun affichage** et n'appelle **jamais** `os.Exit`.

*Piste à explorer avant de coder : `errors.Join` (Go 1.20+) permet de regrouper plusieurs
erreurs en une seule. Est-ce le bon outil ici, ou un `[]error` explicite est-il préférable ?
Argumenter dans un commentaire — les deux réponses sont défendables.*

---

## Défi

**Un type d'erreur porteur de données.**

Jusqu'ici les erreurs ne portent que du texte. Écrire :

```go
type ValidationError struct {
	Line  int
	Field string
	Value string
	Err   error   // l'erreur sous-jacente, éventuellement nil
}
```

**a)** Lui donner une méthode `Error() string` pour qu'elle satisfasse `error`.
*(Les méthodes arrivent au niveau 2, leçon 3 — c'est volontaire : chercher la syntaxe dans la
documentation officielle fait partie du défi.)*

**b)** Lui ajouter `Unwrap() error` et vérifier qu'`errors.Is` traverse bien la structure.

**c)** Côté appelant, utiliser `errors.As` pour récupérer la `*ValidationError` et accéder
à `Line` et `Field` **de façon programmatique**, sans analyser le texte du message.

**d)** Question de conception, à répondre par écrit : `Error()` doit-il être défini sur
`ValidationError` ou sur `*ValidationError` ? Quelles conséquences pour l'appelant ?
*(Indice : quelle est la différence de comportement avec `errors.As` ? C'est le piège
classique, et il annonce le niveau 2.)*

---

## Quiz

1. Quelle est la définition exacte du type `error` ?
2. Que signifie `err == nil` ?
3. Où se place `error` dans la liste des résultats ? Pourquoi cette convention ?
4. Peut-on utiliser les autres résultats quand `err != nil` ?
5. Quelle différence entre `%v` et `%w` dans `fmt.Errorf` ?
6. Pourquoi les messages d'erreur commencent-ils par une minuscule sans point final ?
7. Pourquoi préférer `errors.Is(err, ErrX)` à `err == ErrX` ?
8. Comment signaler qu'on ignore une erreur volontairement ?
9. Dans quels cas `panic` est-il justifié ?
10. Pourquoi ne faut-il pas mettre le mot « erreur » dans un message d'erreur ?
