# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-02/lecon-06/`.

## Exercices faciles

**E1 — `%w` contre `%v`.**
Écrire une chaîne de trois fonctions qui s'appellent et enveloppent l'erreur. Vérifier
qu'`errors.Is` trouve la sentinelle d'origine. Puis remplacer **un seul** `%w` par `%v` au
milieu de la chaîne et constater. Le message affiché a-t-il changé ?

**E2 — `Is` ou `As` ?**
Pour chacun de ces besoins, dire lequel utiliser et pourquoi :
(a) savoir si un fichier n'existe pas ; (b) récupérer le chemin fautif d'une erreur de
système de fichiers ; (c) savoir si un délai a expiré ; (d) connaître le numéro de ligne
d'une erreur d'analyse ; (e) savoir si une requête SQL a violé une contrainte d'unicité.
Puis implémenter (a) et (b) réellement, sur de vrais appels à `os.Open`.

**E3 — Type d'erreur.**
Écrire `HTTPError{Code int; URL string; Err error}` avec `Error()`, `Unwrap()`, et un
constructeur. Écrire un appelant qui utilise `errors.As` pour récupérer le code et décider
s'il faut réessayer (5xx) ou abandonner (4xx).

**E4 — Le piège du récepteur.**
Écrire le même type d'erreur **deux fois** : `Error()` sur la valeur, puis sur le pointeur.
Dans chaque cas, tenter `errors.As` avec `var e MyErr` puis `var e *MyErr`. Dresser le
tableau des quatre combinaisons : lesquelles compilent, lesquelles trouvent l'erreur.
Conclure.

**E5 — `errors.Join`.**
Agréger cinq erreurs dont deux enveloppent la même sentinelle. Vérifier qu'`errors.Is` la
trouve. Afficher le message complet. Que donne `errors.Unwrap` sur le résultat de `Join` ?
*(La réponse surprend — chercher la signature de la méthode utilisée par `Join`.)*

---

## Exercice intermédiaire — `configloader`

Un chargeur de configuration dont les erreurs sont réellement exploitables.

```go
type ConfigError struct {
	Key    string
	Value  string
	Reason string
	Err    error
}

var (
	ErrMissingKey = errors.New("clé manquante")
	ErrBadFormat  = errors.New("format invalide")
)

func Load(path string) (*Config, error)
func LoadFromString(s string) (*Config, error)
```

Format d'entrée : des lignes `clé=valeur`, avec `#` en commentaire.
Clés attendues : `host` (string, obligatoire), `port` (int, 1-65535, obligatoire),
`timeout` (durée en secondes, optionnel, défaut 30), `debug` (booléen, optionnel).

**Contraintes :**
1. **Toutes** les erreurs de configuration sont collectées, pas seulement la première : un utilisateur ne doit pas corriger son fichier ligne par ligne en dix passes.
2. Une erreur d'ouverture de fichier est **fatale** et retournée immédiatement, enveloppée avec le chemin. La distinction fatale/collectée doit être visible dans la structure du code.
3. Chaque `ConfigError` enveloppe soit `ErrMissingKey`, soit `ErrBadFormat`, soit l'erreur d'origine de `strconv`.
4. L'appelant doit pouvoir, avec `errors.As`, récupérer la **liste des clés fautives** — donc réfléchir à ce que `Join` permet et ne permet pas ici.
5. `LoadFromString` existe pour que la logique soit testable **sans fichier**. `Load` ne fait qu'ouvrir et déléguer. *(C'est le même principe `io.Reader` que le projet 1 — pourquoi une `string` plutôt qu'un `io.Reader` ici ? Trancher.)*
6. Aucun `panic`, aucune journalisation dans la bibliothèque.
7. `main` affiche un rapport lisible groupant les erreurs par clé.

*Le vrai sujet de conception est la contrainte 4 : `errors.Join` aplatit tout en une seule
`error`. Comment l'appelant récupère-t-il les `*ConfigError` individuels ? Deux réponses
existent — l'une utilise une méthode peu connue, l'autre change le type de retour. Comparer.*

---

## Défi

**a) `Unwrap() []error` à la main.**
Écrire un type `MultiError` qui agrège des erreurs **sans** utiliser `errors.Join` : avec un
`[]error` interne, `Error()`, et `Unwrap() []error`. Vérifier qu'`errors.Is` et `errors.As`
le traversent correctement. Ajouter une méthode `Errors() []error` pour l'accès direct.
Comparer avec `errors.Join` : qu'apporte la version maison ?

**b) Erreur temporaire.**
Concevoir un mécanisme permettant à un décorateur de réessai (leçon 5, exercice
intermédiaire) de distinguer une erreur **temporaire** d'une erreur **définitive** —
proprement, sans analyser le texte du message.
Trois pistes : une interface `interface{ Temporary() bool }` détectée par assertion, une
sentinelle `ErrTemporary` enveloppée, un type d'erreur avec un champ.
Implémenter les trois, puis dire laquelle retenir. *(Chercher pourquoi
`net.Error.Temporary()` a été **déprécié** en Go 1.18 — la réponse est instructive.)*

**c) Frontière d'API.**
Écrire une fonction `ToPublic(err error) (code int, message string)` qui traduit une erreur
interne en réponse publique : jamais de détail d'implémentation, mais un code exploitable.
Traiter au moins : introuvable, conflit, validation, permission, panne interne.
Puis répondre par écrit : où doit vivre cette fonction dans une architecture en couches, et
que se passe-t-il si une nouvelle erreur interne n'est pas prévue par la traduction ?

---

## Quiz

1. Quelle différence de comportement entre `%w` et `%v` dans `fmt.Errorf` ? Et de texte produit ?
2. Quelle méthode une erreur doit-elle avoir pour participer à une chaîne ?
3. Que fait `errors.Is` exactement ?
4. Que fait `errors.As`, et quelle contrainte pèse sur son second argument ?
5. Quand préférer une sentinelle à un type d'erreur ? Et l'inverse ?
6. Pourquoi les types d'erreur utilisent-ils un récepteur **pointeur** ?
7. Que retourne `errors.Join()` sans argument ? Avec uniquement des `nil` ?
8. Quelle est la signature de la méthode qui permet à `errors.Join` d'être traversée ?
9. Pourquoi ne faut-il pas journaliser **et** retourner la même erreur ?
10. Combien d'informations nouvelles chaque niveau d'enveloppement doit-il ajouter ?
