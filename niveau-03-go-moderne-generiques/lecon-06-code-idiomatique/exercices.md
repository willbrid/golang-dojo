# Leçon 6 — Exercices

Code et réponses écrites dans `mes-solutions/niveau-03/lecon-06/`.

## Exercices faciles

**E1 — Lire les sources.**
Lire **en entier** *Go Code Review Comments* (`go.dev/wiki/CodeReviewComments`). Pour
**cinq** sections au choix, écrire une phrase : la règle, et la leçon de ce cours où elle a
déjà été rencontrée. Pour une section que tu n'avais **jamais** rencontrée, écrire un exemple
qui la viole et sa correction.

**E2 — Renommer.**
Proposer un meilleur nom, en une ligne de justification chacun :

```go
package stringutils
type IStorage interface{ … }
func (self *Cache) GetSize() int
type HttpClientWrapper struct{}
var userIdList []int
func (c *Config) LoadConfigFromFile(configFilePath string) error
package models; type ModelUser struct{}
const MAX_RETRIES = 3
```

**E3 — La ligne d'œil.**
Réécrire sans aucun `else` et sans imbrication au-delà d'un niveau :

```go
func process(path string) (int, error) {
	f, err := os.Open(path)
	if err == nil {
		defer f.Close()
		data, err := io.ReadAll(f)
		if err == nil {
			if len(data) > 0 {
				return bytes.Count(data, []byte("\n")), nil
			} else {
				return 0, errors.New("Empty file")
			}
		} else {
			return 0, err
		}
	} else {
		return 0, err
	}
}
```

Combien de remarques de revue, **hors** structure, trouves-tu encore dans ce code ?

**E4 — Commentaires.**
Écrire les commentaires de documentation du paquet `stats` de la leçon 2. Puis lancer
`go doc ./stats` et `go doc ./stats.Median` depuis la racine du module : le rendu est-il
lisible par quelqu'un qui n'a jamais vu le code ?

**E5 — `bool` en paramètre.**
`func Export(path string, overwrite bool, compress bool) error` s'appelle
`Export("out.csv", true, false)`. Proposer **deux** réécritures, et dire laquelle tu
choisirais si une troisième option arrivait l'an prochain.

---

## Exercice intermédiaire — Revue de code

Le code suivant est soumis en *pull request* à un projet open source. **Rédiger la revue** :
une liste de remarques, chacune avec sa gravité (`bloquant`, `important`, `nit`), le **quoi**,
le **pourquoi**, et une **piste** — sans réécrire le code à la place de l'auteur.

```go
package helpers

import (
	"fmt"
	"os"
	"io/ioutil"
	"strings"
)

var Cache map[string]string

type ConfigManager struct {
	Path string
	data map[string]string
}

func NewConfigManager(path string) *ConfigManager {
	c := ConfigManager{Path: path}
	c.Load()
	return &c
}

func (cm ConfigManager) Load() {
	content, err := ioutil.ReadFile(cm.Path)
	if err != nil {
		panic(err)
	}
	cm.data = make(map[string]string)
	for _, line := range strings.Split(string(content), "\n") {
		parts := strings.Split(line, "=")
		cm.data[parts[0]] = parts[1]
		Cache[parts[0]] = parts[1]
	}
}

func (cm *ConfigManager) GetValue(key string) string {
	if val, ok := cm.data[key]; ok == true {
		return val
	} else {
		fmt.Println("Warning: key " + key + " not found!")
		return ""
	}
}
```

**Attendu :** au moins **4 bugs** (dont deux paniques et une modification perdue), au moins
**6** remarques de conception ou de convention. Classer la revue par gravité décroissante.
Conclure par une phrase d'**appréciation globale** à l'auteur : la revue est lue par un
humain qui a donné de son temps.

---

## Défi — Relire la bibliothèque standard

Choisir **un** de ces fichiers et le lire en entier avec `go doc -src` ou dans
`$(go env GOROOT)/src` :

- `strings/builder.go`
- `slices/sort.go` (seulement les fonctions exportées et leurs commentaires)
- `errors/wrap.go`

Puis écrire (15 à 25 lignes) :

1. Trois conventions de cette leçon que le fichier applique, avec la ligne qui le montre.
2. Une chose qui t'a **surpris** — une construction inconnue, un commentaire qui explique un
   choix, une optimisation.
3. Une chose que tu aurais écrite autrement, et pourquoi les auteurs ont probablement eu
   raison (ou tort).

---

## Quiz

1. Quels sont les deux documents de référence du style Go ?
2. Pourquoi ne discute-t-on jamais de formatage dans une revue Go ?
3. Comment s'écrivent `userId`, `httpUrl`, `jsonApi` en Go idiomatique ?
4. Pourquoi `user.UserService` est-il un mauvais nom ?
5. Par quoi commence un commentaire de documentation ?
6. Quelles sont les trois règles d'un message d'erreur Go ?
7. Que veut dire « journaliser ou retourner, pas les deux » ?
8. Qu'est-ce qu'une « zéro-valeur utile » ? Donner deux exemples de la stdlib.
9. Que signifie `nit:` en tête d'une remarque de revue ?
10. Dans quel ordre traiter les remarques d'une revue ?
