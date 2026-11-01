# Leçon 6 — Code idiomatique

> Contribuer à un projet open source Go, c'est d'abord passer une **revue de code**. Le code
> qui fonctionne mais « ne ressemble pas à du Go » y est renvoyé avec des commentaires laconiques :
> *« receiver name should be a reflection of its identity »*, *« error strings should not be
> capitalized »*, *« accept interfaces, return structs »*. Cette leçon rassemble ces règles,
> explique **d'où elles viennent**, et apprend à relire du code comme un relecteur.

## Objectifs

1. Connaître les conventions que vérifient les relecteurs de projets Go.
2. Comprendre la raison de chacune — pour savoir quand une exception est justifiée.
3. Savoir relire du code et formuler une remarque de revue utile.

## Explication

### Les deux textes de référence

L'équipe Go publie deux documents que tout relecteur connaît :

- **Effective Go** (`go.dev/doc/effective_go`) — le style général du langage. Écrit en 2009,
  il n'a pas été mis à jour pour les génériques ni les modules ; ses principes restent valides.
- **Go Code Review Comments** (`go.dev/wiki/CodeReviewComments`) — la liste des remarques
  les plus fréquentes en revue, avec leur justification.

Les lire **en entier** une fois fait partie de cette leçon (voir exercices). Ce qui suit en
est la synthèse, regroupée par thème, avec les renvois aux leçons où chaque règle est née.

### Formatage : ne pas en discuter

`gofmt` décide. Aucune revue Go ne débat d'indentation ou d'accolades. Un code non formaté est
refusé **avant** lecture. `goimports` (ou `gopls` dans l'éditeur) fait la même chose et gère en
plus les imports : groupe stdlib, ligne vide, groupe externe.

### Nommage

| Règle | Mauvais | Bon | Raison |
|---|---|---|---|
| Courts quand la portée est courte | `for index := range items` | `for i := range items` | la longueur d'un nom suit la distance entre sa déclaration et son usage |
| Pas de répétition du paquet | `user.UserService` | `user.Service` | on lit toujours `paquet.Nom` |
| Acronymes en casse uniforme | `UserId`, `HttpServer`, `parseUrl` | `UserID`, `HTTPServer`, `parseURL` | lisibilité, cohérence avec la stdlib |
| Pas de `Get` pour un accesseur | `u.GetName()` | `u.Name()` | le setter, lui, s'appelle `SetName` |
| Récepteur : 1 à 2 lettres, constant | `func (this *Server)`, `self` | `func (s *Server)` | ce n'est pas un objet au sens Java ; le même nom dans toutes les méthodes |
| Paquet : un mot, minuscule, au singulier | `utils`, `common`, `helpers`, `stringUtils` | `strconv`, `user`, `slug` | un paquet qui s'appelle `utils` ne dit rien de ce qu'il contient (niveau 2, leçon 8) |
| Interface à une méthode : verbe + `er` | `IReader`, `ReaderInterface` | `Reader` | |
| Variable d'erreur | `e`, `error1` | `err` ; sentinelle `ErrNotFound` ; type `NotFoundError` | |

### Erreurs (niveaux 1, leçon 10 et 2, leçon 6)

- Message en **minuscules**, **sans ponctuation finale** : il sera enchaîné
  (`lecture config : ouverture : fichier introuvable`).
- Ajouter le **contexte** à chaque remontée : `fmt.Errorf("chargement de %q : %w", path, err)`.
- Ne pas journaliser **et** retourner : l'erreur serait écrite deux fois. On fait l'un ou l'autre.
- Traiter l'erreur **d'abord**, garder le chemin normal **non indenté** :

```go
// Non idiomatique
if err == nil {
	… 30 lignes …
} else {
	return err
}

// Idiomatique : early return, le « chemin heureux » reste à gauche
if err != nil {
	return err
}
… 30 lignes …
```

- Pas de `panic` pour une erreur attendue (niveau 2, leçon 7).
- `_ = f()` doit être un **choix**, commenté si ce n'est pas évident.

### Commentaires de documentation

- Tout identifiant **exporté** a un commentaire qui commence par **son nom** :
  `// Parse lit une configuration depuis r.` — c'est ce qu'affiche `go doc`.
- Le paquet a un commentaire `// Package config …` dans **un** de ses fichiers (souvent `doc.go`).
- Commenter le **pourquoi**, pas le **quoi** : `i++ // incrémente i` n'apporte rien ;
  `// on saute l'en-tête CSV` en apporte.
- Les phrases sont complètes, avec un point final (contrairement aux messages d'erreur).

### Conception d'API

| Règle | Leçon d'origine |
|---|---|
| **Rendre la zéro-valeur utile** : `var b bytes.Buffer`, `var mu sync.Mutex` fonctionnent sans constructeur | N2 L2 |
| Accepter des interfaces, rendre des types concrets | N2 L4 |
| Interfaces petites, définies côté consommateur | N2 L4 |
| Pas de type pointeur concret retourné dans une variable `error` | N2 L4 |
| Récepteurs tous valeur ou tous pointeur, pour un même type | N2 L3 |
| `(T, bool)` ou `(T, error)` plutôt que des valeurs magiques (`-1`, `""`) | N1 L10 |
| Pas de générique qui n'apporte rien | N3 L1 |
| `context.Context` en **premier** paramètre, jamais dans une struct | N6 L8 |
| Pas d'état global mutable ; `init()` avec parcimonie | N2 L8 |
| Paramètres nombreux → struct d'options ou options fonctionnelles | N1 L11 |

### Taille et forme

- **Ligne d'œil** : le chemin normal d'une fonction se lit en descendant le bord gauche ; les
  cas d'erreur sortent tôt vers la droite.
- **Pas de `else` après un `return`.**
- Une fonction qui ne tient pas sur un écran fait probablement deux choses.
- Préférer `switch` sans expression à une chaîne de `if/else if`.
- Déclarer les variables au plus près de leur usage ; `:=` dans les `if` pour borner la portée
  (`if v, ok := m[k]; ok { … }`).
- Un `bool` en paramètre est souvent illisible à l'appel (`Render(true, false)`) : un type
  nommé ou deux fonctions.

### Quand la règle cède

Ces conventions sont des **valeurs par défaut**. Un relecteur expérimenté accepte une
exception argumentée : un nom long dans une fonction de 200 lignes de calcul scientifique, un
`panic` dans un `MustCompile` appelé à l'initialisation, une interface côté fournisseur quand
elle **est** le produit (`database/sql/driver`). Ce qu'il refuse, c'est l'exception **non
consciente**.

### Formuler une remarque de revue

Une bonne remarque dit **quoi**, **pourquoi**, et propose **une piste** — exactement la
méthode de cette formation :

> ❌ « C'est pas idiomatique. »
>
> ✅ « `GetName` → `Name` : en Go les accesseurs ne prennent pas de préfixe `Get` (voir
> CodeReviewComments, *Getters*). Ça rendra l'appel `u.Name()` cohérent avec la stdlib. »

Et distinguer la gravité : un bug, une violation de convention et une préférence personnelle
ne pèsent pas pareil. Beaucoup de projets préfixent les remarques mineures par `nit:` (*nitpick*)
pour dire « à corriger si tu veux, ça ne bloque pas ».

## Exemple

Un code « qui marche », puis sa version relue. **Lire d'abord la première version, noter
soi-même les problèmes, et seulement ensuite lire la seconde.**

```go
// Version d'origine — fonctionne, mais une revue la renverrait.
package utils

import "errors"
import "strings"
import "fmt"

type UserStruct struct {
	userId   int
	UserName string
}

func (this *UserStruct) GetUserName() string {
	return this.UserName
}

func FindUser(users []UserStruct, name string) (UserStruct, error) {
	var found bool = false
	var result UserStruct
	for index := 0; index < len(users); index++ {
		if strings.ToLower(users[index].UserName) == strings.ToLower(name) {
			found = true
			result = users[index]
		}
	}
	if found == true {
		return result, nil
	} else {
		fmt.Println("user not found")
		return result, errors.New("User not found.")
	}
}
```

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

// ErrNotFound est retournée quand aucun utilisateur ne correspond.
var ErrNotFound = errors.New("utilisateur introuvable")

// User est un utilisateur de l'annuaire.
type User struct {
	id   int
	name string
}

// Name rend le nom de l'utilisateur.
func (u *User) Name() string { return u.name }

// FindUser rend le premier utilisateur dont le nom correspond à name,
// sans tenir compte de la casse.
func FindUser(users []User, name string) (User, error) {
	for _, u := range users {
		if strings.EqualFold(u.name, name) {
			return u, nil
		}
	}
	return User{}, fmt.Errorf("%q : %w", name, ErrNotFound)
}

func main() {
	users := []User{{1, "Awa"}, {2, "Inès"}, {3, "awa"}}

	u, err := FindUser(users, "AWA")
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(u.id, u.Name())

	_, err = FindUser(users, "Zoé")
	fmt.Println(err, errors.Is(err, ErrNotFound))
}
```

Sortie :

```
1 Awa
"Zoé" : utilisateur introuvable true
```

## Explication du code

| Remarque de revue | Gravité |
|---|---|
| **Bug** : la boucle d'origine ne s'arrête pas au premier trouvé et rend donc le **dernier** — `"awa"` (id 3) au lieu de `"Awa"`. Le `return` dans la boucle corrige et clarifie. | bloquant |
| **Bug** : `strings.ToLower(a) == strings.ToLower(b)` alloue deux chaînes et échoue sur certains caractères Unicode ; `strings.EqualFold` est fait pour ça. | bloquant |
| Fonction de bibliothèque qui **affiche** (`fmt.Println`) en plus de retourner l'erreur : journaliser **ou** retourner. | bloquant |
| Paquet `utils` → un nom qui dit ce qu'il contient (`user`, `directory`). | important |
| `UserStruct` → `User` ; `userId` → `id` (ou `ID` si exporté). | important |
| `this` → `u` ; `GetUserName` → `Name`. | important |
| Message d'erreur `"User not found."` → minuscule, sans point, avec la valeur cherchée. Sentinelle exportée pour `errors.Is`. | important |
| `var found bool = false`, `if found == true`, `else` après `return` : bruit. | nit |
| Trois `import` séparés → un bloc groupé ; `gofmt`/`goimports` l'auraient fait. | nit |
| `for index := 0; index < len(users); index++` → `for _, u := range users`. | nit |
| Commentaires de documentation absents sur les identifiants exportés. | important |

Remarquer l'ordre : **d'abord les bugs**. Une revue qui commence par des `nit:` en ratant le bug
de la boucle a échoué.

## Erreurs fréquentes

1. **Importer le style d'un autre langage** : `this`, `GetX`, `IFoo`, classes de base
   simulées par embedding, `utils`.
2. **Bégaiement** : `config.ConfigLoader`, `http.HTTPServer`.
3. **Corriger le style en ratant le bug** pendant une revue.
4. **Commentaires qui paraphrasent le code.**
5. **Over-engineering** : interfaces, fabriques et génériques pour une seule implémentation.
6. **Réponses défensives en revue** au lieu d'argumenter ou de corriger.

## Bonnes pratiques Go

- `gofmt` (ou `goimports`) à chaque sauvegarde, `go vet` avant chaque commit.
- Lire **Effective Go** et **Code Review Comments** une fois en entier, puis y revenir quand
  une remarque de revue cite une section.
- Lire du code de la bibliothèque standard : `go doc -src strings.EqualFold`. C'est le
  meilleur corpus de Go idiomatique qui existe.
- En revue : bugs d'abord, puis conception, puis conventions, puis `nit:`.
- *Clear is better than clever* : le code clair l'emporte sur le code astucieux.

## Ce que je dois retenir

- `gofmt` n'est pas négociable.
- Noms courts pour les portées courtes ; pas de bégaiement ; acronymes en casse uniforme ;
  pas de `Get` ; récepteur d'une ou deux lettres.
- Erreurs : minuscules, contexte, `%w`, journaliser **ou** retourner, early return.
- Doc : commence par le nom de l'identifiant ; commenter le pourquoi.
- Zéro-valeur utile, interfaces petites côté consommateur, types concrets en retour.
- En revue : quoi, pourquoi, piste ; bugs d'abord.

---

➡️ [Exercices](exercices.md)
