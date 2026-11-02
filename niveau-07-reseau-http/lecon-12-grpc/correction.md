# Leçon 12 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Générer et lire

Le serveur implémente `InventoryServiceServer`, une interface avec `GetItem`, `WatchStock` et une
méthode non exportée `mustEmbedUnimplementedInventoryServiceServer()` — qui **oblige** à embarquer la
struct `Unimplemented…` (on ne peut pas implémenter une méthode non exportée d'un autre paquet). Cette
struct fournit des implémentations de toutes les méthodes qui rendent `codes.Unimplemented`. Les
accesseurs générés commencent par `if x != nil` : `GetSku()` sur un pointeur nil rend `""` au lieu de
paniquer — on peut enchaîner `req.GetItem().GetSku()` sans vérification intermédiaire.

## E2 — Erreur ordinaire

Le client reçoit `codes.Unknown`, avec le texte de l'erreur. Un client qui réessaie seulement
`Unavailable` (transitoire) ne sait pas classer `Unknown` : soit il ne réessaie jamais (perte de
résilience), soit il réessaie tout (y compris des erreurs définitives). Et le texte de l'erreur interne
part chez le client — la fuite de la leçon 5.

## E3 — Échéance transmise

Le client reçoit `codes.DeadlineExceeded` au bout de 50 ms. Le serveur voit son `ctx` annulé au même
moment : l'échéance du client est transmise dans l'en-tête `grpc-timeout` et reconstruite côté serveur.
Vérification : faire signaler l'annulation sur un channel par la méthode (comme le `Store` de la leçon 7),
ou journaliser `ctx.Err()` dans un intercepteur.

## E4 — Évolution du contrat

- Nouveau champ `description = 5` : l'ancien client **ignore** le champ inconnu (protobuf conserve même
  les champs inconnus en mémoire) ; rien ne casse. Le nouveau serveur recevant un message d'un ancien
  client voit `description` vide.
- `stock` passé de `int32` à `string` sous le même numéro : les données binaires d'un côté sont
  interprétées avec le mauvais type de l'autre — erreur de décodage ou, selon les types, valeurs
  **fausses sans erreur**. C'est le pire des cas.
- La bonne évolution : un **nouveau** champ (`string stock_label = 6;`), et l'ancien marqué
  `reserved 4; reserved "stock";` une fois supprimé : le compilateur `protoc` refuse alors toute
  réutilisation future du numéro ou du nom. `buf breaking` détecte ces ruptures automatiquement en CI.

## E5 — `grpcurl`

```go
reflection.Register(srv)
```

`grpcurl -plaintext localhost:9090 list` affiche `inventory.v1.InventoryService` (et le service de
réflexion) ; `grpcurl -plaintext -d '{"sku":"KB-01"}' localhost:9090
inventory.v1.InventoryService/GetItem` rend l'article en JSON. Sur un service exposé, la réflexion
**publie** toute la surface de l'API (méthodes internes comprises) : une aide précieuse pour un
attaquant. On l'active en développement, ou derrière une authentification.

## Exercice intermédiaire — Un intercepteur d'authentification

```go
func Auth(valid map[[32]byte]string) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, h grpc.UnaryHandler) (any, error) {
		md, _ := metadata.FromIncomingContext(ctx)
		vals := md.Get("authorization")             // clés de métadonnées en minuscules
		if len(vals) != 1 || !strings.HasPrefix(vals[0], "Bearer ") {
			return nil, status.Error(codes.Unauthenticated, "jeton requis")
		}
		sum := sha256.Sum256([]byte(strings.TrimPrefix(vals[0], "Bearer ")))
		user, ok := valid[sum]                      // empreinte : pas de jeton brut stocké (leçon 8)
		if !ok {
			return nil, status.Error(codes.Unauthenticated, "jeton invalide")
		}
		return h(context.WithValue(ctx, userKey{}, user), req)
	}
}

func Recover(log io.Writer) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, h grpc.UnaryHandler) (resp any, err error) {
		defer func() {
			if v := recover(); v != nil {
				fmt.Fprintf(log, "%s panique : %v\n%s", info.FullMethod, v, debug.Stack())
				err = status.Error(codes.Internal, "erreur interne")
			}
		}()
		return h(ctx, req)
	}
}

srv := grpc.NewServer(grpc.ChainUnaryInterceptor(Logging(log), Recover(log), Auth(tokens)))
```

- L'**indexation par empreinte SHA-256** évite la comparaison caractère par caractère d'un jeton secret :
  une recherche dans une map sur l'empreinte ne révèle rien d'exploitable sur le jeton (leçon 8, E4).
- Ordre : journalisation à l'extérieur, récupération ensuite, authentification au plus près du handler —
  comme pour HTTP (leçon 3). Une panique dans l'authentification elle-même est aussi récupérée.
- Les intercepteurs **de flux** (`grpc.StreamServerInterceptor`) ont une signature différente : il faut
  envelopper le `grpc.ServerStream` pour modifier son contexte.

## Défi — Flux bidirectionnel

```go
func (s *Server) Reserve(stream grpc.BidiStreamingServer[inventoryv1.ReserveRequest, inventoryv1.ReserveResult]) error {
	for {
		req, err := stream.Recv()
		if errors.Is(err, io.EOF) {
			return nil                                // le client a fait CloseSend
		}
		if err != nil {
			return err                                // annulation, coupure
		}
		ok := s.take(req.GetSku(), req.GetQuantity()) // vérifier ET décrémenter sous un seul verrou
		if err := stream.Send(&inventoryv1.ReserveResult{Sku: req.GetSku(), Accepted: ok}); err != nil {
			return err
		}
	}
}

func (s *Server) take(sku string, n int32) bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	it, ok := s.items[sku]
	if !ok || it.Stock < n {
		return false
	}
	it.Stock -= n
	return true
}
```

- `take` est la leçon 6 du niveau 6 : la décision et l'action sous le **même** verrou. Avec
  `Available()` puis `Take()` séparés, le test des 10 clients accepterait parfois plus de 5 réservations.
- Côté client, la goroutine d'envoi termine par `stream.CloseSend()` ; la boucle de réception lit jusqu'à
  `io.EOF`. Les deux sont attendues (`errgroup`, N6 L9).
- Le test : 10 clients en parallèle, chacun 1 réservation sur un stock de 5 ; compter les `Accepted` —
  exactement 5, et le stock final à 0, sous `-race`.

## Quiz

1. Son numéro.
2. Compatibles : ajouter un champ, renommer un champ, supprimer un champ en réservant son numéro.
   Interdits : réutiliser un numéro, changer le type d'un champ existant.
3. `*.pb.go` (messages) et `*_grpc.pb.go` (client et interface serveur).
4. Pour que le serveur compile encore quand le contrat gagne des méthodes, qui rendent alors
   `Unimplemented`.
5. `status.Error(codes.NotFound, "…")` ; `status.Code(err)`.
6. Elle est transmise et devient l'échéance du contexte du serveur.
7. Unaire, flux serveur, flux client, bidirectionnel.
8. Une fonction qui enveloppe chaque appel : l'équivalent gRPC d'un middleware.
9. À tester client et serveur réels par une connexion en mémoire, sans réseau.
10. REST pour les API publiques et les navigateurs ; gRPC entre services internes, pour le contrat typé,
    la performance et le streaming.
