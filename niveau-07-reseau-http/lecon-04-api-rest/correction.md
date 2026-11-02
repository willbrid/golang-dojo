# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Critiquer une API

| Avant | Problème | Après |
|---|---|---|
| `GET /getAllUsers` | verbe dans l'URL | `GET /v1/users` (paginé) |
| `POST /user/create` | verbe, singulier | `POST /v1/users` |
| `GET /user?id=42` | identifiant en paramètre, singulier | `GET /v1/users/42` |
| `POST /user/42/delete` | verbe, mauvaise méthode | `DELETE /v1/users/42` |
| `POST /updateUserEmail` | verbe, identifiant dans le corps, opération trop spécifique | `PATCH /v1/users/42` avec `{"email": "…"}` |
| `GET /users/42/orders/7/items/3/reviews` | imbrication trop profonde, couplage à toute la chaîne | `GET /v1/order-items/3/reviews` (ou `/v1/reviews?item=3`) |

## E2 — Idempotence

- (a) `PUT` complet : **idempotent** — remplacer deux fois par la même chose donne le même état.
- (b) `POST` de dépôt : **non** — deux envois, deux dépôts. Clé d'idempotence (`Idempotency-Key`)
  mémorisée par le serveur avec la réponse.
- (c) `{"increment": 1}` : **non** — chaque appel ajoute 1. Remplacer par une valeur cible (`{"value":
  10}`) si c'est possible, ou une clé d'idempotence.
- (d) `{"done": true}` : **idempotent** — c'est une affectation, pas un incrément.

Règle : une **affectation** est idempotente, une **opération relative** (ajouter, incrémenter,
créer) ne l'est pas.

## E3 — Absent ou zéro

Avec des types valeur, `{"title":"x"}` décode `Done` à `false` : la tâche terminée redevient **non
terminée**, alors que le client voulait seulement changer le titre. Test :

```go
call(t, a, "PATCH", "/v1/tasks/2", `{"done":true}`)
call(t, a, "PATCH", "/v1/tasks/2", `{"title":"renommée"}`)
_, body := call(t, a, "GET", "/v1/tasks/2", "")
if !strings.Contains(body, `"done":true`) {
	t.Errorf("le PATCH du titre a remis done à false : %s", body)
}
```

## E4 — Pagination instable

Par décalage : page 1 (`offset=0`) = tâches 1 et 2 ; on supprime la 1 ; page 2 (`offset=2`) = la liste
est maintenant `[2, 3, 4…]`, donc la page 2 commence à… la **4**. La tâche 3 n'est jamais vue. (Une
insertion produit l'effet inverse : un élément vu deux fois.) Avec le curseur (`cursor=2` : « après
l'id 2 »), la page 2 commence bien à la 3, quelles que soient les suppressions.

## E5 — Curseur opaque

```go
func encodeCursor(id int) string { return base64.RawURLEncoding.EncodeToString([]byte(strconv.Itoa(id))) }
```

Un curseur opaque dit au client « ne fabriquez pas ce jeton, renvoyez-le tel quel ». Le serveur peut
alors en changer le contenu (un identifiant + une date de tri, un tri par nom…) sans casser personne.
Un curseur lisible (`cursor=2`) invite les clients à le construire eux-mêmes, et le figerait. Ce n'est
**pas** une protection : base64 se décode trivialement ; si le curseur doit être infalsifiable, on le
**signe** (HMAC).

## Exercice intermédiaire — Une API de bibliothèque

Tableau de routes possible :

| Méthode | Chemin | Corps | Succès | Erreurs |
|---|---|---|---|---|
| GET | `/v1/books` | — | 200 | 400 (filtre invalide) |
| POST | `/v1/books` | `{title, author_id, isbn}` | 201 + Location | 400, 409 (ISBN existant), 422 |
| GET | `/v1/books/{id}` | — | 200 | 404 |
| PATCH | `/v1/books/{id}` | champs partiels | 200 | 404, 422 |
| DELETE | `/v1/books/{id}` | — | 204 | 409 (emprunté) |
| GET | `/v1/authors/{id}` | — | 200 | 404 |
| POST | `/v1/loans` | `{book_id, member_id}` | 201 | 404, 409 (déjà emprunté) |
| PATCH | `/v1/loans/{id}` | `{"returned": true}` | 200 | 404, 409 (déjà rendu) |
| GET | `/v1/members/{id}/loans?active=true` | — | 200 | 404 |

« Rendre » est modélisé comme un **changement d'état** de l'emprunt (`PATCH /v1/loans/{id}`), ce qui
garde l'historique. L'emprunt est une ressource à part entière, pas un champ du livre : un livre a
plusieurs emprunts au fil du temps.

Idempotence de `POST /v1/loans` :

```go
type idemEntry struct {
	status int
	body   []byte
}

key := r.Header.Get("Idempotency-Key")
if key != "" {
	a.mu.Lock()
	if e, ok := a.idem[key]; ok {
		a.mu.Unlock()
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(e.status)
		w.Write(e.body)                        // rejouer la réponse mémorisée
		return
	}
	a.mu.Unlock()
}
// … créer l'emprunt, encoder la réponse dans un buffer, la mémoriser sous key, puis l'envoyer.
```

Points délicats : deux requêtes **simultanées** avec la même clé (il faut réserver la clé sous verrou
avant de traiter, sinon les deux créent un emprunt) ; la durée de conservation des clés (24 h
typiquement) ; et la même clé envoyée avec un **corps différent** (erreur 422, c'est un bug client).

## Défi — Faire évoluer sans casser

1. Ajouter `due` facultatif est **compatible** — à condition que les clients **ignorent les champs
   inconnus**. Un client écrit avec `DisallowUnknownFields` côté décodage des **réponses** casserait :
   c'est pour cela qu'on réserve cette rigueur aux **entrées** du serveur.
2. Renommer `done` est incompatible. Stratégie raisonnable : `/v2/tasks` avec `status` ; `/v1/tasks`
   continue de servir `done`, calculé à partir de `status` (`done = status == "done"`). Le stockage est
   unique, seule la **représentation** diffère : deux structs de sortie (`TaskV1`, `TaskV2`) et deux
   fonctions de conversion depuis le modèle interne. Quand un client `v1` écrit `done: false` sur une
   tâche `doing`, il faut décider (garder `doing`) et le documenter.
3. `Deprecation` (RFC 9745) annonce que la ressource est dépréciée ; `Sunset` (RFC 8594) donne la date
   prévue de retrait. Un client bien écrit les **journalise** ou les remonte en alerte, pour que l'équipe
   migre avant la date.

## Quiz

1. L'URL désigne une ressource ; l'action est portée par la méthode HTTP.
2. 201, un en-tête `Location` vers la ressource, et la ressource créée.
3. GET, HEAD, PUT, DELETE (et OPTIONS) ; parce que le réseau perd des réponses et que les clients
   réessaient.
4. Pour qu'un réessai après une réponse perdue ne soit pas vu comme une erreur.
5. Avec une clé d'idempotence fournie par le client, mémorisée avec la réponse.
6. Avec des champs pointeurs dans la struct de décodage (`nil` = absent).
7. Lenteur sur de grandes tables, et doublons ou omissions quand les données changent entre deux pages.
8. Pour pouvoir ajouter des métadonnées (curseur, total) sans casser les clients.
9. Ajouter des champs de réponse, des paramètres facultatifs, des routes — pas renommer, supprimer, ni
   changer un type ou un statut.
10. Chemin : simple, visible, facile à tester et à router ; en-tête : URL stables, mais plus difficile à
    tester, à mettre en cache et à déboguer.
