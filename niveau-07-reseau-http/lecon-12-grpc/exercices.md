# Leçon 12 — Exercices

Code dans `mes-solutions/niveau-07/lecon-12/`. Installer `protoc` (paquet `protobuf-compiler` de la
distribution) et les deux générateurs (`go install …protoc-gen-go@latest` et `…protoc-gen-go-grpc@latest`).

## Exercices faciles

**E1 — Générer et lire.**
Générer le code de l'exemple. Ouvrir `inventory_grpc.pb.go` : quelle interface le serveur doit-il
implémenter ? Que contient `UnimplementedInventoryServiceServer` ? Pourquoi les accesseurs `GetSku()`
fonctionnent-ils sur un `*GetItemRequest` nil ?

**E2 — Erreur ordinaire.**
Faire rendre à `GetItem` un `errors.New("introuvable")` au lieu d'un `status.Error`. Quel code reçoit le
client ? Pourquoi est-ce un problème pour un client qui veut réessayer seulement les erreurs
transitoires ?

**E3 — Échéance transmise.**
Ajouter à `GetItem` une attente de 200 ms qui respecte `ctx`. Appeler avec un contexte de 50 ms. Quel code
reçoit le client ? Le serveur a-t-il vu l'annulation ? Comment le vérifier ?

**E4 — Évolution du contrat.**
Ajouter un champ `string description = 5;` à `Item`, régénérer, et faire dialoguer un **ancien** client
(code généré avant la modification) avec le nouveau serveur. Que voit-il ? Puis changer le type du champ
`stock` de `int32` en `string` en gardant le numéro 4 : que se passe-t-il ? Comment `reserved` évite-t-il
ce piège ?

**E5 — `grpcurl`.**
Activer la réflexion sur le serveur (`google.golang.org/grpc/reflection`, `reflection.Register(srv)`),
lancer le serveur sur un vrai port, et l'interroger avec `grpcurl -plaintext localhost:9090 list`, puis
appeler `GetItem`. Pourquoi ne pas activer la réflexion sur un service exposé ?

---

## Exercice intermédiaire — Un intercepteur d'authentification

1. Le client envoie un jeton dans les **métadonnées** (`metadata.AppendToOutgoingContext(ctx,
   "authorization", "Bearer …")`).
2. Un intercepteur **unaire et de flux** côté serveur lit le jeton (`metadata.FromIncomingContext`),
   le compare en temps constant (leçon 8) à une liste de jetons hachés, et rend `Unauthenticated` sinon.
   L'identité est placée dans le contexte (comme le middleware de la leçon 8).
3. Un intercepteur de **récupération de panique** qui rend `Internal` et journalise la pile (N6 L12).
4. Les composer avec `grpc.ChainUnaryInterceptor` dans le bon ordre (leçon 3).
5. Tests `bufconn` : sans jeton, mauvais jeton, bon jeton, méthode qui panique.

---

## Défi — Flux bidirectionnel

Ajouter au service une méthode `rpc Reserve(stream ReserveRequest) returns (stream ReserveResult);` :
le client envoie des demandes de réservation (sku, quantité) au fil de l'eau, le serveur répond à chacune
(acceptée ou refusée selon le stock), dans l'ordre.

1. Le stock est partagé entre plusieurs clients simultanés : le protéger (N6 L6) et garantir qu'il ne
   devient jamais négatif (« vérifier puis agir » sous un même verrou).
2. Côté client : une goroutine envoie, une autre reçoit (N6) ; `CloseSend` signale la fin des envois.
3. Le serveur s'arrête proprement si le client annule son contexte.
4. Test : 10 clients réservent en parallèle 1 unité d'un article en stock de 5 ; exactement 5 réservations
   acceptées, sous `-race`.

---

## Quiz

1. Qu'est-ce qui identifie un champ dans le format binaire protobuf ?
2. Quels changements d'un `.proto` sont compatibles ? Lesquels sont interdits ?
3. Quels fichiers génèrent `protoc-gen-go` et `protoc-gen-go-grpc` ?
4. Pourquoi embarquer `Unimplemented…Server` ?
5. Comment rendre une erreur `NotFound` ? Comment le client la lit-il ?
6. Que devient l'échéance du contexte du client côté serveur ?
7. Quelles sont les quatre formes d'appel gRPC ?
8. Qu'est-ce qu'un intercepteur ?
9. À quoi sert `bufconn` ?
10. Quand préférer REST, quand préférer gRPC ?
