# Projet 3 — `bookshelf`, une API REST testée de bout en bout

**À réaliser après le niveau 7.** Durée indicative : 15 à 25 heures.
Écrire le code dans `mes-solutions/projets/projet-03/`.

---

## Objectif pédagogique

Livrer une API HTTP **complète et professionnelle** avec la seule bibliothèque standard : routage 1.22+,
JSON strict, modèle d'erreur unique, middlewares, authentification, pagination, délais, arrêt gracieux —
et une suite de tests qui couvre chaque route et chaque cas d'erreur. Le stockage reste **en mémoire**,
derrière une interface : le projet 4 le remplacera par PostgreSQL **sans toucher** au reste.

---

## Spécifications

Une API de gestion de bibliothèque personnelle partagée : des utilisateurs, leurs livres, leurs avis.

```
POST   /v1/users                         inscription               → 201
POST   /v1/sessions                      connexion (cookie)        → 204
DELETE /v1/sessions                      déconnexion               → 204
GET    /v1/me                            utilisateur courant       → 200

GET    /v1/books?author=&q=&limit=&cursor=   liste paginée          → 200
POST   /v1/books                         ajouter un livre (auth.)  → 201 + Location
GET    /v1/books/{id}                    détail                    → 200
PATCH  /v1/books/{id}                    modifier (propriétaire)   → 200
DELETE /v1/books/{id}                    supprimer (propriétaire)  → 204

POST   /v1/books/{id}/reviews            noter (auth., 1 par utilisateur et par livre) → 201
GET    /v1/books/{id}/reviews            liste paginée              → 200

GET    /healthz                          vivant                    → 200
GET    /readyz                           prêt à recevoir du trafic → 200 / 503
```

### Comportements exigés

| # | Exigence |
|---|---|
| S1 | JSON en entrée et en sortie ; corps bornés (64 Kio) ; champs inconnus refusés ; `Content-Type` vérifié (415). |
| S2 | Toutes les erreurs au format *problem details* (RFC 9457, `application/problem+json`), y compris les 404 et 405 du routeur. |
| S3 | Validation : titre (1–200 caractères), auteur (1–100), ISBN-13 valide (chiffre de contrôle vérifié), année entre 1450 et l'année courante, note de 1 à 5. Toutes les erreurs d'un même corps sont rendues ensemble, champ par champ (422). |
| S4 | Un ISBN ne peut figurer qu'une fois par utilisateur (409). Un utilisateur ne note un livre qu'une fois (409). |
| S5 | `PATCH` partiel (champs absents inchangés) ; seul le propriétaire modifie ou supprime son livre — un autre utilisateur reçoit **404** (on ne révèle pas l'existence, N7 L5). |
| S6 | Listes paginées par **curseur** opaque, `limit` de 1 à 100 (défaut 20), triées par date de création puis identifiant. |
| S7 | Mots de passe hachés (bcrypt ou argon2id) ; sessions par cookie `HttpOnly`, `Secure`, `SameSite=Lax` ; sessions expirées après 24 h. |
| S8 | Limite de débit sur `POST /v1/sessions` : 5 essais par minute par email et 30 par IP (429 + `Retry-After`). |
| S9 | Chaque réponse porte un `X-Request-ID` ; chaque requête est journalisée (méthode, motif de route, statut, durée, identifiant) ; les 5xx avec le détail de l'erreur. |
| S10 | Une panique dans un handler donne un 500 *problem details* et ne fait pas tomber le serveur. |
| S11 | Serveur avec délais et limites (N7 L9), arrêt gracieux sur `SIGINT`/`SIGTERM` (N7 L7), `/readyz` à 503 pendant l'arrêt. |
| S12 | Configuration par variables d'environnement (`ADDR`, `BCRYPT_COST`, `SESSION_TTL`…) validées au démarrage : une valeur invalide empêche le démarrage avec un message clair. |

---

## Contraintes

1. **Bibliothèque standard** + `golang.org/x/crypto` (hachage des mots de passe). Pas de routeur ni de
   framework externe.
2. Architecture en paquets sous `internal/` : `domain` (types et règles, **sans** `net/http`), `store`
   (une **interface** définie côté consommateur + une implémentation en mémoire), `httpapi` (handlers,
   middlewares, traduction des erreurs), `auth`.
3. Les handlers ont la forme `func(w, r) error` avec un adaptateur unique (N7 L5).
4. Le stockage en mémoire est sûr en concurrence et passe une **suite de tests de contrat** écrite contre
   l'interface : le projet 4 réutilisera **la même suite** pour l'implémentation PostgreSQL.
5. Toutes les dépendances (horloge, générateur d'identifiants et de jetons, journal) sont injectées.
6. `go test -race ./...` passe ; couverture ≥ 80 % sur `domain` et `httpapi`, chaque ligne non couverte
   justifiable (N5 L4).
7. `go vet`, `go fix -diff`, `golangci-lint` (configuration de N5 L8) silencieux.

---

## Difficultés glissées volontairement

- **D1.** Deux requêtes simultanées ajoutent le même ISBN pour le même utilisateur. Les deux passent-elles
  la vérification de doublon ? (N6 L6 : « vérifier puis agir ».)
- **D2.** La pagination par curseur « date de création puis identifiant » : comment encoder un curseur à
  deux composantes, et que faire si deux livres ont la même date à la nanoseconde ?
- **D3.** Un utilisateur supprime un livre qui a des avis. Que deviennent-ils ? Décider, documenter, tester.
- **D4.** `PATCH` avec `{"year": null}` : est-ce « effacer l'année » ou une erreur ? Comment distinguer
  `null` d'un champ absent en Go ? (N4 L6, N7 L4.)
- **D5.** Un client envoie `X-Request-ID: ` suivi de 10 000 caractères. Et `X-Forwarded-For` falsifié
  pour contourner la limite de débit (N7 L9).
- **D6.** La recherche `q` dans les titres doit-elle être sensible à la casse et aux accents (« Etranger »
  doit-il trouver « L'Étranger ») ? Décider et documenter (N4 L1).
- **D7.** L'arrêt gracieux est déclenché pendant une requête de connexion (bcrypt, 250 ms). Se termine-t-elle ?
- **D8.** Le format JSON d'un livre en sortie et la struct de stockage : pourquoi ne pas utiliser la même
  struct ? (Pensez au hachage du mot de passe dans un `User`, et au projet 4.)

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.**

```
projet-03/
├── go.mod
├── cmd/bookshelf/main.go        configuration, assemblage, signal, Run
└── internal/
    ├── domain/                  Book, Review, User ; Validate ; erreurs (ErrNotFound, ErrConflict, *ValidationError)
    ├── store/
    │   ├── store.go             interfaces consommées par le domaine/les handlers
    │   ├── memory.go            implémentation en mémoire
    │   └── storetest/suite.go   tests de contrat réutilisables : func Run(t *testing.T, newStore func() Store)
    ├── auth/                    hachage, sessions, middleware RequireUser
    └── httpapi/
        ├── server.go            routes, NewServer (délais), Run (arrêt gracieux)
        ├── books.go, reviews.go, users.go
        ├── errors.go            traduction domaine → problem details
        └── middleware.go        request ID, journal, recover, limite de débit
```

L'idée directrice : `httpapi` dépend de `domain` et des **interfaces** de `store` ; `main` est le seul
endroit qui choisit l'implémentation. Le paquet `storetest` est la clé du projet 4.

---

## Tests à écrire

- `domain` : table de validation (ISBN valides et invalides, bornes de longueur et d'année).
- `storetest` : création, lecture, doublons, pagination (curseur stable sous insertion), suppression,
  concurrence (D1) — exécutée contre l'implémentation mémoire.
- `httpapi` : un test « conversation » par ressource (N7 L4), chaque statut d'erreur, problem details
  partout, propriété (404 pour un autre utilisateur), limite de débit (avec horloge injectée), panique → 500.
- Middleware de journal : contenu exact d'une ligne (horloge fausse).
- Arrêt gracieux : requête en cours terminée, `/readyz` à 503 (N7 L7).
- Un fuzzing de la validation d'ISBN et du décodage du curseur (N5 L7).

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 12 spécifications sont-elles satisfaites, testées par `httptest` et vérifiables avec `curl` ? |
| **C'est correct** | Les 8 difficultés traitées ou documentées ; `-race` silencieux ; aucune fuite d'information dans les erreurs. |
| **C'est idiomatique** | Domaine sans HTTP, interfaces côté consommateur, handlers `func(w, r) error`, middlewares `Handler → Handler`, contexte propagé, aucun état global. |
| **Prêt pour la production** | Délais, limites, arrêt gracieux, sondes de santé, journaux exploitables, configuration validée, sécurité des mots de passe et des cookies. |

Rends ton code même incomplet : une conception discutée vaut mieux qu'un projet abandonné.
