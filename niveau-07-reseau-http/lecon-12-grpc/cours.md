# Leçon 12 — RPC et gRPC

> REST et JSON conviennent aux API publiques : lisibles, universelles, testables avec `curl`. Entre
> services internes, d'autres critères pèsent : un **contrat typé** vérifié à la compilation des deux
> côtés, un format binaire compact, le **streaming** dans les deux sens. C'est ce qu'apporte gRPC —
> né chez Google, et très répandu dans l'écosystème Go (Kubernetes, etcd, beaucoup de bases de données
> l'utilisent). Cette leçon le met en œuvre, et dit quand **ne pas** l'utiliser.

## Objectifs

1. Décrire un service dans un fichier `.proto` et générer le code Go.
2. Implémenter un serveur gRPC unaire et en flux, et l'appeler.
3. Utiliser les codes d'état gRPC, le contexte et les intercepteurs.
4. Choisir entre REST et gRPC.

## Explication

### Le contrat d'abord : Protocol Buffers

Un fichier `.proto` décrit les **messages** et le **service** :

```proto
message Item {
  string sku = 1;
  int64 price_cents = 3;
}

service InventoryService {
  rpc GetItem(GetItemRequest) returns (Item);
}
```

Les **numéros de champ** (`= 1`, `= 3`) sont l'identité de chaque champ dans le format binaire — pas
le nom. D'où les règles d'évolution : on peut **ajouter** des champs, **renommer** un champ sans rien
casser, mais **jamais réutiliser** un numéro ni changer le type d'un champ existant (on marque les
anciens numéros `reserved`). Un champ absent vaut sa zéro-valeur, comme en Go.

### Générer le code

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
protoc --go_out=. --go_opt=module=exemple \
       --go-grpc_out=. --go-grpc_opt=module=exemple \
       proto/inventory/v1/inventory.proto
```

Deux fichiers sont produits : `inventory.pb.go` (les messages, avec leurs accesseurs `GetSku()` qui
tolèrent un pointeur nil) et `inventory_grpc.pb.go` (l'interface serveur à implémenter et le client).
Ils commencent par `// Code generated … DO NOT EDIT.` (N4 L8) et se **commitent**. L'outil `buf`
remplace avantageusement `protoc` dans les projets réels (configuration, *lint* et détection des
changements incompatibles du `.proto`).

### Le serveur

On implémente l'interface générée en **embarquant** `UnimplementedInventoryServiceServer` : si le
`.proto` gagne une méthode, le serveur compile encore et répond `Unimplemented` pour elle — la
compatibilité ascendante par la composition (N2 L5).

```go
srv := grpc.NewServer(grpc.UnaryInterceptor(logging))
inventoryv1.RegisterInventoryServiceServer(srv, impl)
srv.Serve(lis)                 // lis : un net.Listener (leçon 10)
```

### Les erreurs : des codes, pas des statuts HTTP

Une méthode rend une `error` construite avec `status.Error(codes.X, msg)`. Les codes correspondent aux
cas de la leçon 5 : `InvalidArgument` (400/422), `NotFound` (404), `AlreadyExists` / `Aborted` (409),
`PermissionDenied` (403), `Unauthenticated` (401), `ResourceExhausted` (429), `Unavailable` (503,
réessayable), `DeadlineExceeded` (504), `Internal` (500). Côté client : `status.Code(err)`. Même règle
qu'en HTTP : la traduction erreur métier → code se fait **à un seul endroit**, et aucun détail interne ne
sort.

### Le contexte traverse gRPC

Le client passe un `context.Context` à chaque appel ; son **échéance** est transmise au serveur dans la
requête, et son annulation interrompt l'appel des deux côtés. Tout ce que la leçon 7 a dit du contexte
de bout en bout s'applique, sans code supplémentaire.

### Le streaming

| Forme | `.proto` | Usage |
|---|---|---|
| unaire | `rpc F(Req) returns (Resp)` | le cas courant |
| flux serveur | `returns (stream Resp)` | abonnement, gros résultats en morceaux |
| flux client | `rpc F(stream Req) returns (Resp)` | envoi par lots, téléversement |
| bidirectionnel | `stream` des deux côtés | conversation, synchronisation |

Côté serveur, `stream.Send` envoie ; côté client, `stream.Recv` reçoit jusqu'à `io.EOF`, la fin normale
du flux.

### Les intercepteurs

L'équivalent des middlewares (leçon 3) : une fonction qui enveloppe chaque appel, pour la
journalisation, l'authentification (lire les métadonnées, l'équivalent des en-têtes), les métriques,
le traçage (niveau 12), la récupération de panique. Unaires et de flux, côté serveur et côté client.

### Tester sans réseau : `bufconn`

`google.golang.org/grpc/test/bufconn` fournit un `net.Listener` **en mémoire** : le vrai serveur et le
vrai client, sans port ni réseau. C'est l'équivalent gRPC de `httptest`.

### REST ou gRPC ?

| Critère | REST + JSON | gRPC |
|---|---|---|
| clients | navigateurs, tiers, tout langage, `curl` | services internes ; navigateur seulement via un proxy (gRPC-Web) |
| contrat | à documenter (OpenAPI) | **le `.proto`**, vérifié à la compilation |
| performance | texte, plus volumineux | binaire, compact, HTTP/2 |
| streaming | SSE, WebSocket | natif, dans les deux sens |
| débogage | lisible partout | outils dédiés (`grpcurl`) |

Règle raisonnable : **REST pour ce qui est public, gRPC entre services internes** quand le contrat typé
ou le streaming apportent quelque chose. Les deux peuvent coexister (`grpc-gateway` expose un service
gRPC en REST).

## Exemple

Le contrat :

```proto
syntax = "proto3";

package inventory.v1;

option go_package = "exemple/gen/inventory/v1;inventoryv1";

// Le contrat : un service, ses méthodes, ses messages.
service InventoryService {
  // Unaire : une requête, une réponse.
  rpc GetItem(GetItemRequest) returns (Item);
  // Flux serveur : une requête, plusieurs réponses.
  rpc WatchStock(WatchStockRequest) returns (stream StockEvent);
}

message GetItemRequest {
  string sku = 1;
}

message Item {
  string sku = 1;
  string name = 2;
  int64 price_cents = 3;
  int32 stock = 4;
}

message WatchStockRequest {
  string sku = 1;
  int32 max_events = 2;
}

message StockEvent {
  string sku = 1;
  int32 stock = 2;
}
```

Après la génération (commande ci-dessus), le serveur :

```go
// inventory/server.go
// Package inventory implémente le service gRPC décrit dans proto/inventory/v1.
package inventory

import (
	"context"
	"fmt"
	"io"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"

	inventoryv1 "exemple/gen/inventory/v1"
)

type Server struct {
	inventoryv1.UnimplementedInventoryServiceServer // compatibilité ascendante : méthodes futures « non implémentées »
	items map[string]*inventoryv1.Item
}

func NewServer() *Server {
	return &Server{items: map[string]*inventoryv1.Item{
		"KB-01": {Sku: "KB-01", Name: "clavier", PriceCents: 4990, Stock: 3},
	}}
}

func (s *Server) GetItem(ctx context.Context, req *inventoryv1.GetItemRequest) (*inventoryv1.Item, error) {
	if req.GetSku() == "" {
		return nil, status.Error(codes.InvalidArgument, "sku requis") // l'équivalent gRPC d'un 400
	}
	it, ok := s.items[req.GetSku()]
	if !ok {
		return nil, status.Errorf(codes.NotFound, "article %q introuvable", req.GetSku())
	}
	return it, nil
}

// WatchStock envoie plusieurs messages sur un même appel, jusqu'à max_events ou l'annulation du client.
func (s *Server) WatchStock(req *inventoryv1.WatchStockRequest, stream grpc.ServerStreamingServer[inventoryv1.StockEvent]) error {
	it, ok := s.items[req.GetSku()]
	if !ok {
		return status.Errorf(codes.NotFound, "article %q introuvable", req.GetSku())
	}
	stock := it.GetStock()
	for i := range req.GetMaxEvents() {
		if err := stream.Send(&inventoryv1.StockEvent{Sku: it.GetSku(), Stock: stock - i}); err != nil {
			return err
		}
		select {
		case <-time.After(10 * time.Millisecond):
		case <-stream.Context().Done(): // le contexte traverse gRPC comme HTTP (leçon 7)
			return stream.Context().Err()
		}
	}
	return nil
}

// Logging est un intercepteur unaire : le middleware de gRPC (leçon 3).
func Logging(out io.Writer) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
		resp, err := handler(ctx, req)
		fmt.Fprintf(out, "%s %s\n", info.FullMethod, status.Code(err))
		return resp, err
	}
}
```

Et ses tests, avec `bufconn` :

```go
// inventory/server_test.go
package inventory

import (
	"bytes"
	"context"
	"errors"
	"io"
	"net"
	"testing"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/status"
	"google.golang.org/grpc/test/bufconn"

	inventoryv1 "exemple/gen/inventory/v1"
)

// newClient démarre le serveur sur une connexion EN MÉMOIRE (bufconn) : pas de port, pas de réseau.
func newClient(t *testing.T, log io.Writer) inventoryv1.InventoryServiceClient {
	t.Helper()
	lis := bufconn.Listen(1 << 20)
	srv := grpc.NewServer(grpc.UnaryInterceptor(Logging(log)))
	inventoryv1.RegisterInventoryServiceServer(srv, NewServer())
	go srv.Serve(lis)
	t.Cleanup(srv.Stop)

	conn, err := grpc.NewClient("passthrough:///bufnet",
		grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) { return lis.DialContext(ctx) }),
		grpc.WithTransportCredentials(insecure.NewCredentials()), // en test seulement : TLS en production (leçon 11)
	)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { conn.Close() })
	return inventoryv1.NewInventoryServiceClient(conn)
}

func TestGetItem(t *testing.T) {
	var log bytes.Buffer
	c := newClient(t, &log)
	it, err := c.GetItem(t.Context(), &inventoryv1.GetItemRequest{Sku: "KB-01"})
	if err != nil || it.GetName() != "clavier" || it.GetPriceCents() != 4990 {
		t.Fatalf("GetItem = %v, %v", it, err)
	}
	for _, tc := range []struct {
		sku  string
		want codes.Code
	}{{"", codes.InvalidArgument}, {"XX", codes.NotFound}} {
		_, err := c.GetItem(t.Context(), &inventoryv1.GetItemRequest{Sku: tc.sku})
		if got := status.Code(err); got != tc.want {
			t.Errorf("sku %q : code %v, attendu %v", tc.sku, got, tc.want)
		}
	}
	want := "/inventory.v1.InventoryService/GetItem OK\n" +
		"/inventory.v1.InventoryService/GetItem InvalidArgument\n" +
		"/inventory.v1.InventoryService/GetItem NotFound\n"
	if log.String() != want {
		t.Errorf("journal :\n%s", log.String())
	}
}

func TestWatchStock(t *testing.T) {
	c := newClient(t, io.Discard)
	stream, err := c.WatchStock(t.Context(), &inventoryv1.WatchStockRequest{Sku: "KB-01", MaxEvents: 3})
	if err != nil {
		t.Fatal(err)
	}
	var stocks []int32
	for {
		ev, err := stream.Recv()
		if errors.Is(err, io.EOF) { // fin normale du flux
			break
		}
		if err != nil {
			t.Fatal(err)
		}
		stocks = append(stocks, ev.GetStock())
	}
	if len(stocks) != 3 || stocks[0] != 3 || stocks[2] != 1 {
		t.Errorf("événements = %v", stocks)
	}
}
```

Le `go.mod` du module :

```
module exemple

go 1.27

require (
	google.golang.org/grpc v1.82.0
	google.golang.org/protobuf v1.36.11
)

require (
	golang.org/x/net v0.53.0 // indirect
	golang.org/x/sys v0.43.0 // indirect
	golang.org/x/text v0.36.0 // indirect
	google.golang.org/genproto/googleapis/rpc v0.0.0-20260414002931-afd174a4e478 // indirect
)
```

`go test -race ./...` passe (vérifié avec grpc v1.82.0, protobuf v1.36.11, `protoc` 3.21).

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `package inventory.v1` et le chemin `proto/inventory/v1/` | La version fait partie du nom : `v2` coexistera, comme `/v1/` en REST (leçon 4). |
| `go_package` | Le chemin d'import et le nom du paquet Go générés. |
| `price_cents` en `int64` | L'argent en entiers (N4 L3), même en protobuf. |
| `UnimplementedInventoryServiceServer` embarqué | Le serveur compile encore si le contrat gagne une méthode. |
| `status.Error(codes.NotFound, …)` | Des codes typés ; le client les lit avec `status.Code(err)`. |
| `stream.Send` et `stream.Context()` | Un flux respecte aussi l'annulation du client. |
| `Logging` | Un intercepteur unaire : `info.FullMethod` donne `/paquet.Service/Méthode`. |
| `bufconn` + `passthrough:///bufnet` | Le vrai client parle au vrai serveur, en mémoire. |
| `insecure.NewCredentials()` | Sans TLS : **seulement** en test. En production, `credentials.NewTLS` avec la configuration de la leçon 11, souvent en mTLS. |
| `stream.Recv` jusqu'à `io.EOF` | La fin normale d'un flux serveur. |

## Erreurs fréquentes

1. **Réutiliser un numéro de champ** ou changer son type : corruption silencieuse des messages.
2. **Ne pas embarquer `Unimplemented…Server`.**
3. **Rendre une `error` ordinaire** : le client reçoit `Unknown`, sans code exploitable.
4. **Envoyer le texte d'erreurs internes** dans le message de statut.
5. **Appels sans échéance** côté client.
6. **gRPC pour une API publique** consommée par des navigateurs et des tiers.
7. **Désactiver TLS en production** (`insecure`).
8. **Ne pas commiter le code généré**, ou le modifier à la main.

## Bonnes pratiques Go

- Le `.proto` est le contrat : versionné dans le chemin du paquet, numéros jamais réutilisés (`reserved`).
- Code généré commité ; `buf` pour la génération et la détection des ruptures.
- Erreurs métier traduites en codes gRPC à un seul endroit.
- Contexte avec échéance sur chaque appel ; TLS (mTLS) entre services.
- Intercepteurs pour les préoccupations transverses ; tests avec `bufconn`.

## Ce que je dois retenir

- Contrat en Protocol Buffers ; les **numéros** de champ sont l'identité, jamais réutilisés.
- `protoc` + `protoc-gen-go` + `protoc-gen-go-grpc` génèrent messages, client et interface serveur.
- Erreurs : `status.Error(codes.X, …)`, lues par `status.Code(err)`.
- Le contexte (échéance, annulation) traverse l'appel.
- Unaire, flux serveur, flux client, bidirectionnel ; intercepteurs = middlewares.
- REST pour le public, gRPC entre services internes.

---

➡️ [Exercices](exercices.md)
