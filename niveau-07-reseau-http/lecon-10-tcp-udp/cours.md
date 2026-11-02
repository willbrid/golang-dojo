# Leçon 10 — TCP et UDP

> Sous HTTP, il y a TCP — un flux d'octets fiable et ordonné, **sans aucune notion de message**.
> Bases de données, files de messages, caches, protocoles de jeux, le serveur clé-valeur du projet 5 :
> tous définissent leur propre façon de découper ce flux en messages. Cette leçon apprend à le faire
> correctement (*framing*), à écrire des protocoles binaires, et présente UDP et le DNS.

## Objectifs

1. Écrire un serveur et un client TCP avec `net.Listen`, `Accept`, `Dial`, et des délais.
2. Découper un flux en messages : préfixe de longueur et `io.ReadFull`.
3. Encoder un protocole binaire avec `encoding/binary`, en bornant tout ce qui vient du réseau.
4. Connaître UDP et la résolution DNS avec contexte.

## Explication

### TCP : un flux, pas des messages

```go
ln, err := net.Listen("tcp", "127.0.0.1:7070")   // ":0" : port choisi par le système
conn, err := ln.Accept()                           // bloque jusqu'à un client
conn, err := net.Dial("tcp", "127.0.0.1:7070")     // côté client ; DialContext avec un contexte
```

Un `net.Conn` est un `io.Reader` + `io.Writer` + `Close` (N4 L5). Le point fondamental : **TCP ne
conserve pas les frontières des écritures**. Deux `Write` de 10 octets peuvent arriver en un `Read` de
20 octets, ou en trois `Read` de 7, 7 et 6. Un programme qui suppose « un `Write` = un `Read` »
fonctionne en local (tout arrive d'un coup) et casse sur un vrai réseau. Il faut un **framing**.

### Trois façons de découper

| Méthode | Exemple | Remarque |
|---|---|---|
| **délimiteur** | une commande par ligne (`\n`) — projet 5, SMTP, Redis | simple, lisible ; le délimiteur ne doit pas apparaître dans les données (ou être échappé) ; `bufio.Scanner` avec une limite (64 Kio par défaut) |
| **préfixe de longueur** | 4 octets de longueur puis la charge | données binaires quelconques ; lecture exacte avec `io.ReadFull` |
| longueur fixe | des enregistrements de 128 octets | rare |

Pour le préfixe de longueur :

```go
var hdr [5]byte
io.ReadFull(r, hdr[:])                         // EXACTEMENT 5 octets, quel que soit le découpage
n := binary.BigEndian.Uint32(hdr[1:])
if n > MaxPayload { … }                         // TOUJOURS borner avant d'allouer
payload := make([]byte, n)
io.ReadFull(r, payload)
```

`io.ReadFull` rend `io.EOF` si **rien** n'a été lu (fin propre entre deux messages) et
`io.ErrUnexpectedEOF` si la connexion s'est fermée **au milieu** d'un message — deux situations à
distinguer.

La borne `MaxPayload` est vitale : sans elle, quatre octets `FF FF FF FF` envoyés par n'importe qui
font allouer 4 Gio au serveur — la bombe de décompression de N5 L7, version réseau.

### `encoding/binary`

`binary.BigEndian.PutUint32(b, v)` / `Uint32(b)` encodent les entiers dans un ordre d'octets fixé.
Les protocoles réseau utilisent par convention le **gros-boutiste** (*big-endian*, « ordre réseau »).
`binary.Read`/`binary.Write` encodent des structs entières, mais par réflexion — plus lents ; pour un
protocole chaud, on encode à la main ou avec `binary.Append…` (Go 1.23).

### Délais : `SetDeadline`

Une connexion n'a **aucun délai** par défaut. `conn.SetDeadline(t)` (ou `SetReadDeadline` /
`SetWriteDeadline`) fixe une **échéance absolue** : au-delà, les opérations en cours et futures
échouent avec une erreur dont `Timeout()` est vrai (`errors.Is(err, os.ErrDeadlineExceeded)`). Pour un
délai d'**inactivité**, on la **repousse** avant chaque lecture.

Le délai est aussi le moyen d'**interrompre** une lecture bloquée depuis une autre goroutine (avec
`SetReadDeadline(time.Now())`), tout comme `Close`. C'est ainsi qu'on relie un `context` à une
connexion : `context.AfterFunc(ctx, func() { conn.Close() })` (Go 1.21).

### Le serveur type

Une boucle `Accept`, une goroutine par connexion, un arrêt par contexte qui **ferme le listener**
(pour débloquer `Accept`) et **ferme les connexions** (pour débloquer les `Read`), puis un `WaitGroup`
pour attendre leur fin. C'est la structure de l'exemple — et celle attendue au projet 5.

### UDP

UDP envoie des **datagrammes** indépendants : pas de connexion, pas de garantie de livraison ni d'ordre,
mais les **frontières sont conservées** (un envoi = une réception, ou rien). Utilisé pour le DNS, la
vidéo, les jeux, les métriques (StatsD), là où la latence prime et où une perte est acceptable.

```go
pc, _ := net.ListenPacket("udp", "127.0.0.1:9000")
buf := make([]byte, 1500)                       // taille typique d'un paquet (MTU)
n, addr, _ := pc.ReadFrom(buf)                  // un datagramme entier, et son expéditeur
pc.WriteTo(buf[:n], addr)                       // réponse
```

Un datagramme plus grand que le buffer est **tronqué** sans erreur sur la plupart des systèmes. Et
comme l'adresse source d'un paquet UDP se falsifie facilement, un serveur UDP qui répond plus gros qu'il
ne reçoit peut servir d'**amplificateur** dans une attaque.

### DNS

```go
r := &net.Resolver{}
addrs, err := r.LookupHost(ctx, "go.dev")       // avec contexte : délai et annulation
mx, err := r.LookupMX(ctx, "exemple.fr")
```

Go a son propre résolveur DNS (pur Go) et peut aussi utiliser celui du système (via cgo) selon la
plateforme ; `GODEBUG=netdns=go` ou `=cgo` force l'un ou l'autre. Toujours passer un contexte : une
résolution lente bloque sinon tout le `Dial`.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : frame/frame.go
// Package frame implémente un protocole binaire à messages préfixés par leur longueur :
//
//	+--------+-----------------+----------------------+
//	| type   | longueur (u32,  | charge utile         |
//	| 1 octet| gros-boutiste)  | « longueur » octets  |
//	+--------+-----------------+----------------------+
package frame

import (
	"bufio"
	"context"
	"encoding/binary"
	"errors"
	"fmt"
	"io"
	"net"
	"strings"
	"sync"
	"time"
)

const (
	TypeEcho  byte = 1
	TypeUpper byte = 2
	TypeError byte = 0xFF
)

// MaxPayload borne la taille annoncée : sans elle, un en-tête de 4 octets
// pourrait réclamer 4 Gio d'allocation (N5 L7 : la bombe).
const MaxPayload = 1 << 20

var ErrTooLarge = errors.New("message trop grand")

type Message struct {
	Type    byte
	Payload []byte
}

// Write écrit un message complet. Un seul Write sur un bufio.Writer vidé ensuite :
// l'en-tête et la charge partent ensemble.
func Write(w *bufio.Writer, m Message) error {
	if len(m.Payload) > MaxPayload {
		return ErrTooLarge
	}
	var hdr [5]byte
	hdr[0] = m.Type
	binary.BigEndian.PutUint32(hdr[1:], uint32(len(m.Payload)))
	if _, err := w.Write(hdr[:]); err != nil {
		return err
	}
	if _, err := w.Write(m.Payload); err != nil {
		return err
	}
	return w.Flush()
}

// Read lit exactement un message. io.ReadFull : un Read TCP peut rendre moins que demandé (N4 L5).
func Read(r io.Reader) (Message, error) {
	var hdr [5]byte
	if _, err := io.ReadFull(r, hdr[:]); err != nil {
		return Message{}, err // io.EOF propre si la connexion se ferme entre deux messages
	}
	n := binary.BigEndian.Uint32(hdr[1:])
	if n > MaxPayload {
		return Message{}, fmt.Errorf("longueur annoncée %d : %w", n, ErrTooLarge)
	}
	payload := make([]byte, n)
	if _, err := io.ReadFull(r, payload); err != nil {
		return Message{}, fmt.Errorf("charge tronquée : %w", err) // io.ErrUnexpectedEOF
	}
	return Message{Type: hdr[0], Payload: payload}, nil
}

// Serve accepte des connexions jusqu'à l'annulation de ctx ; une goroutine par connexion.
func Serve(ctx context.Context, ln net.Listener, idle time.Duration) error {
	var wg sync.WaitGroup
	defer wg.Wait()
	stop := context.AfterFunc(ctx, func() { ln.Close() }) // débloque Accept à l'annulation
	defer stop()
	for {
		conn, err := ln.Accept()
		if err != nil {
			if ctx.Err() != nil {
				return nil
			}
			return err
		}
		wg.Go(func() { handle(ctx, conn, idle) })
	}
}

func handle(ctx context.Context, conn net.Conn, idle time.Duration) {
	defer conn.Close()
	stop := context.AfterFunc(ctx, func() { conn.Close() }) // débloque Read à l'arrêt
	defer stop()
	r, w := bufio.NewReader(conn), bufio.NewWriter(conn)
	for {
		conn.SetDeadline(time.Now().Add(idle)) // délai d'inactivité, renouvelé à chaque message
		m, err := Read(r)
		if err != nil {
			if errors.Is(err, ErrTooLarge) {
				Write(w, Message{Type: TypeError, Payload: []byte(err.Error())})
			}
			return // EOF, délai, ou protocole violé : on ferme
		}
		var reply Message
		switch m.Type {
		case TypeEcho:
			reply = m
		case TypeUpper:
			reply = Message{Type: TypeUpper, Payload: []byte(strings.ToUpper(string(m.Payload)))}
		default:
			reply = Message{Type: TypeError, Payload: fmt.Appendf(nil, "type inconnu %d", m.Type)}
		}
		if err := Write(w, reply); err != nil {
			return
		}
	}
}
```

```go
// Fichier : frame/frame_test.go
package frame

import (
	"bufio"
	"bytes"
	"context"
	"encoding/binary"
	"errors"
	"io"
	"net"
	"testing"
	"time"
)

func startServer(t *testing.T) string {
	t.Helper()
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		t.Fatal(err)
	}
	ctx, cancel := context.WithCancel(context.Background())
	done := make(chan error, 1)
	go func() { done <- Serve(ctx, ln, 2*time.Second) }()
	t.Cleanup(func() {
		cancel()
		if err := <-done; err != nil {
			t.Errorf("Serve : %v", err)
		}
	})
	return ln.Addr().String()
}

func TestRoundTrip(t *testing.T) {
	conn, err := net.Dial("tcp", startServer(t))
	if err != nil {
		t.Fatal(err)
	}
	defer conn.Close()
	r, w := bufio.NewReader(conn), bufio.NewWriter(conn)
	for _, tc := range []struct {
		in, want Message
	}{
		{Message{TypeEcho, []byte("bonjour")}, Message{TypeEcho, []byte("bonjour")}},
		{Message{TypeUpper, []byte("été")}, Message{TypeUpper, []byte("ÉTÉ")}},
		{Message{TypeEcho, nil}, Message{TypeEcho, []byte{}}},
		{Message{42, []byte("x")}, Message{TypeError, []byte("type inconnu 42")}},
	} {
		if err := Write(w, tc.in); err != nil {
			t.Fatal(err)
		}
		got, err := Read(r)
		if err != nil || got.Type != tc.want.Type || !bytes.Equal(got.Payload, tc.want.Payload) {
			t.Errorf("%v → %v %q (%v), attendu %v %q", tc.in.Type, got.Type, got.Payload, err, tc.want.Type, tc.want.Payload)
		}
	}
}

// Un octet à la fois : le framing doit tenir, quel que soit le découpage TCP.
func TestByteByByte(t *testing.T) {
	var buf bytes.Buffer
	w := bufio.NewWriter(&buf)
	Write(w, Message{TypeEcho, []byte("fragmenté")})
	got, err := Read(iotestOneByte{&buf})
	if err != nil || string(got.Payload) != "fragmenté" {
		t.Errorf("%q, %v", got.Payload, err)
	}
}

type iotestOneByte struct{ r io.Reader }

func (o iotestOneByte) Read(p []byte) (int, error) { return o.r.Read(p[:min(1, len(p))]) }

func TestHugeLength(t *testing.T) {
	var hdr [5]byte
	hdr[0] = TypeEcho
	binary.BigEndian.PutUint32(hdr[1:], 0xFFFFFFFF) // annonce 4 Gio
	if _, err := Read(bytes.NewReader(hdr[:])); !errors.Is(err, ErrTooLarge) {
		t.Errorf("err = %v, attendu ErrTooLarge sans allocation", err)
	}
}

func TestTruncated(t *testing.T) {
	var hdr [5]byte
	binary.BigEndian.PutUint32(hdr[1:], 10)
	_, err := Read(bytes.NewReader(append(hdr[:], "abc"...)))
	if !errors.Is(err, io.ErrUnexpectedEOF) {
		t.Errorf("err = %v, attendu io.ErrUnexpectedEOF", err)
	}
}

func FuzzRead(f *testing.F) {
	f.Add([]byte{1, 0, 0, 0, 3, 'a', 'b', 'c'})
	f.Fuzz(func(t *testing.T, data []byte) {
		Read(bytes.NewReader(data)) // ne doit jamais paniquer ni allouer plus que MaxPayload
	})
}
```

`go test -race ./frame` passe, et `go test -run '^$' -fuzz FuzzRead -fuzztime 10s ./frame` ne trouve
rien : aucune entrée ne fait paniquer `Read` ni allouer plus de 1 Mio.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| Le schéma en commentaire de paquet | Un protocole binaire se **documente** octet par octet ; c'est son contrat. |
| `io.ReadFull` pour l'en-tête et la charge | Correct quel que soit le découpage TCP — `TestByteByByte` le prouve en livrant un octet à la fois. |
| `n > MaxPayload` **avant** `make` | `TestHugeLength` envoie une longueur de 4 Gio : refus, sans allocation. |
| `io.ErrUnexpectedEOF` | `TestTruncated` : une charge coupée est distinguée d'une fin propre. |
| `bufio.Writer` + `Flush` | En-tête et charge partent en un seul envoi réseau, sans copie intermédiaire. |
| `conn.SetDeadline` renouvelé à chaque message | Délai d'**inactivité** : un client silencieux est déconnecté. |
| `context.AfterFunc(ctx, ln.Close)` | L'annulation débloque `Accept` ; `Serve` rend alors `nil`. |
| `context.AfterFunc(ctx, conn.Close)` | Et débloque les `Read` en cours dans chaque connexion. |
| `defer wg.Wait()` dans `Serve` | `Serve` ne rend la main qu'une fois toutes les connexions fermées : aucune goroutine ne fuit. |
| `startServer` avec `t.Cleanup` | Chaque test arrête son serveur et vérifie que l'arrêt est propre. |
| `FuzzRead` | Toute fonction qui lit le réseau se fuzze (N5 L7). |

## Erreurs fréquentes

1. **Supposer qu'un `Read` rend un message entier.**
2. **`conn.Read(buf)` au lieu de `io.ReadFull`** pour un en-tête de taille fixe.
3. **Allouer la longueur annoncée sans borne.**
4. **Aucun délai** sur les connexions.
5. **Ne pas débloquer `Accept` et `Read` à l'arrêt** : le serveur ne s'arrête jamais.
6. **Ordre des octets non spécifié** (ou `NativeEndian` dans un protocole réseau).
7. **UDP avec un buffer trop petit** : troncature silencieuse.
8. **Résolution DNS sans contexte.**

## Bonnes pratiques Go

- Documenter le format octet par octet ; gros-boutiste.
- `io.ReadFull` pour les lectures de taille connue ; `bufio.Scanner` borné pour les protocoles en lignes.
- Borner toute longueur, tout compte, toute taille annoncés par le pair.
- `SetDeadline` systématique, renouvelé pour l'inactivité.
- Arrêt : fermer listener et connexions via le contexte, attendre les goroutines.
- Fuzzer le décodeur ; tester avec un lecteur qui livre un octet à la fois.

## Ce que je dois retenir

- TCP = un **flux** : il faut un framing (délimiteur ou préfixe de longueur).
- `io.ReadFull` : `io.EOF` (rien lu) vs `io.ErrUnexpectedEOF` (message coupé).
- Toute longueur reçue est **bornée** avant allocation.
- `SetDeadline` (absolu) ; `Close` ou une échéance immédiate pour interrompre.
- UDP : datagrammes, frontières conservées, aucune garantie.
- DNS : `net.Resolver` avec contexte.

---

➡️ [Exercices](exercices.md)
