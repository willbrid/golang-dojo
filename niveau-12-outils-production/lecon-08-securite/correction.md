# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — `govulncheck` dans un script

Le flux `-format json` est une **suite** d'objets JSON, sans tableau qui les englobe. Sur le programme du
cours, on compte 1 objet `config`, 1 `SBOM`, 2 `progress`, 192 `osv` (les fiches de la base) et 7
`finding`. Un `finding` est **atteignable** quand le premier maillon de sa trace désigne une **fonction**. Les
autres ne désignent qu'un paquet importé ou un module requis :

```
GO-2021-0113 trace[0] = {module: golang.org/x/text, package: golang.org/x/text/language, function: Parse}   ← atteignable
GO-2021-0113 trace[0] = {module: golang.org/x/text, package: golang.org/x/text/language}                    ← paquet importé
GO-2021-0113 trace[0] = {module: golang.org/x/text}                                                         ← module requis
```

```go
// Fichier : cmd/vulnreport/main.go
// Commande vulnreport : résume la sortie de govulncheck -format json (lue sur l'entrée standard).
//
//	govulncheck -format json ./... | vulnreport
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"os"
	"slices"
)

type frame struct {
	Module, Version, Package, Function, Receiver string
	Position                                     *struct {
		Filename     string
		Line, Column int
	}
}

type message struct {
	OSV *struct {
		ID, Summary string
	} `json:"osv"`
	Finding *struct {
		OSV          string  `json:"osv"`
		FixedVersion string  `json:"fixed_version"`
		Trace        []frame `json:"trace"`
	} `json:"finding"`
}

func main() {
	n, err := report(os.Stdin, os.Stdout)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(2)
	}
	if n > 0 {
		os.Exit(3) // comme govulncheck : au moins une vulnérabilité atteignable
	}
}

// report écrit une ligne par vulnérabilité atteignable et renvoie leur nombre.
func report(r io.Reader, w io.Writer) (int, error) {
	dec := json.NewDecoder(r) // le flux est une suite d'objets JSON, sans tableau englobant
	summaries := map[string]string{}
	var reachable []string
	seen := map[string]bool{}
	for {
		var m message
		if err := dec.Decode(&m); errors.Is(err, io.EOF) {
			break
		} else if err != nil {
			return 0, err
		}
		if m.OSV != nil {
			summaries[m.OSV.ID] = m.OSV.Summary
		}
		f := m.Finding
		// Atteignable : le premier maillon de la trace est une FONCTION (sinon : paquet ou module seulement).
		if f == nil || len(f.Trace) == 0 || f.Trace[0].Function == "" || seen[f.OSV] {
			continue
		}
		seen[f.OSV] = true
		vuln, caller := f.Trace[0], f.Trace[len(f.Trace)-1]
		where := ""
		if caller.Position != nil {
			where = fmt.Sprintf("%s:%d", caller.Position.Filename, caller.Position.Line)
		}
		reachable = append(reachable, fmt.Sprintf("%s  %s@%s → %s  %s.%s appelle %s.%s  (%s)",
			f.OSV, vuln.Module, vuln.Version, f.FixedVersion, caller.Package, caller.Function,
			vuln.Package, vuln.Function, where))
	}
	slices.Sort(reachable)
	for _, l := range reachable {
		fmt.Fprintln(w, l)
	}
	for id := range seen {
		fmt.Fprintf(w, "  %s : %s\n", id, summaries[id])
	}
	return len(reachable), nil
}
```

```go
// Fichier : cmd/vulnreport/main_test.go
package main

import (
	"strings"
	"testing"
)

// Un extrait réel de govulncheck -format json, réduit à l'essentiel.
const stream = `{"osv":{"id":"GO-2021-0113","summary":"Out-of-bounds read in golang.org/x/text/language"}}
{"osv":{"id":"GO-2022-1059","summary":"Denial of service via crafted Accept-Language header in golang.org/x/text/language"}}
{"finding":{"osv":"GO-2022-1059","fixed_version":"v0.3.8","trace":[{"module":"golang.org/x/text","version":"v0.3.5","package":"golang.org/x/text/language"}]}}
{"finding":{"osv":"GO-2021-0113","fixed_version":"v0.3.7","trace":[{"module":"golang.org/x/text","version":"v0.3.5","package":"golang.org/x/text/language","function":"Parse","position":{"filename":"language/parse.go","line":33}},{"module":"example.com/svc","package":"example.com/svc/cmd/svc","function":"main","position":{"filename":"cmd/svc/main.go","line":11}}]}}
`

func TestReport(t *testing.T) {
	var out strings.Builder
	n, err := report(strings.NewReader(stream), &out)
	if err != nil {
		t.Fatal(err)
	}
	if n != 1 || !strings.Contains(out.String(), "GO-2021-0113") || strings.Contains(out.String(), "GO-2022-1059") {
		t.Fatalf("n=%d\n%s", n, out.String())
	}
}
```

```
$ govulncheck -format json ./... | vulnreport
GO-2021-0113  golang.org/x/text@v0.3.5 → v0.3.7  example.com/svc/cmd/svc.main appelle golang.org/x/text/language.Parse  (cmd/svc/main.go:11)
  GO-2021-0113 : Out-of-bounds read in golang.org/x/text/language
$ echo $?
3
```

`json.Decoder` lit naturellement un flux d'objets successifs : on appelle `Decode` en boucle jusqu'à
`io.EOF`. Les fiches `osv` arrivent **avant** les `finding` qui les citent, ce qui permet de garder leur résumé
dans une map au passage.

## E2 — La vulnérabilité transitive

Le module `a` dépend de `x/text@v0.3.5` et appelle `language.Parse`. `svc` dépend de `a`, et n'importe pas
`x/text` lui-même.

```
$ govulncheck ./...
Vulnerability #1: GO-2021-0113
    Found in: golang.org/x/text@v0.3.5
    Fixed in: golang.org/x/text@v0.3.7
      #1: main.go:10:33: svc.main calls a.Lang, which calls language.Parse
Your code is affected by 1 vulnerability from 1 module.
$ go mod why -m golang.org/x/text
# golang.org/x/text
example.com/svc
example.com/a
golang.org/x/text/language
$ go mod graph
example.com/svc example.com/a@v0.0.0
example.com/svc golang.org/x/text@v0.3.5
example.com/a@v0.0.0 golang.org/x/text@v0.3.5
[…]
```

`govulncheck` suit les appels **à travers** les dépendances : la trace montre `svc.main → a.Lang →
language.Parse`. `go mod why` donne le chemin d'**import**, et `go mod graph` le chemin des
**exigences** entre modules.

La correction se fait **depuis `svc`**, sans attendre `a` :

```
$ go get golang.org/x/text@v0.3.8
go: upgraded golang.org/x/text v0.3.5 => v0.3.8
$ grep x/text go.mod
require golang.org/x/text v0.3.8 // indirect
$ govulncheck ./...
No vulnerabilities found.
```

La sélection de version minimale (N2 L9) prend la **plus haute** des versions exigées par le graphe. En
exigeant v0.3.8 dans `svc`, on l'impose à `a`. C'est sûr tant que `x/text` respecte la compatibilité au
sein de v0. Les tests de `svc` le vérifient. On signale ensuite le problème aux mainteneurs de `a`.

## E3 — La rotation

```go
// Fichier : internal/auth/auth.go
// Package auth : des jetons d'API vérifiés en temps constant, avec rotation.
package auth

import (
	"crypto/rand"
	"crypto/sha256"
	"crypto/subtle"
	"sync/atomic"
)

// Verifier accepte plusieurs jetons valides (l'ancien et le nouveau pendant une rotation).
// Le jeu de jetons se remplace d'un bloc, sans verrou ni course (leçon 5).
type Verifier struct{ hashes atomic.Pointer[[][32]byte] }

func NewVerifier(tokens ...string) *Verifier {
	v := &Verifier{}
	v.Set(tokens...)
	return v
}

// Set remplace l'ensemble des jetons valides.
func (v *Verifier) Set(tokens ...string) {
	hs := make([][32]byte, len(tokens))
	for i, t := range tokens {
		hs[i] = sha256.Sum256([]byte(t))
	}
	v.hashes.Store(&hs)
}

// Check compare avec TOUS les jetons, sans s'arrêter au premier qui correspond : le temps ne dit
// pas lequel a été reconnu (seulement combien il y en a, ce qui n'est pas un secret).
func (v *Verifier) Check(got string) bool {
	h := sha256.Sum256([]byte(got))
	ok := 0
	for _, want := range *v.hashes.Load() {
		ok |= subtle.ConstantTimeCompare(h[:], want[:])
	}
	return ok == 1
}

// NewToken renvoie un jeton aléatoire, imprévisible (crypto/rand, jamais math/rand).
func NewToken() string { return rand.Text() }

// CheckNaive : la comparaison ordinaire, qui s'arrête au premier octet différent (cours).
func CheckNaive(got, want string) bool { return got == want }
```

```go
// Fichier : internal/auth/rotate_test.go
package auth

import (
	"fmt"
	"math/rand/v2"
	"testing"
	"time"
)

func TestRotation(t *testing.T) {
	const a, b = "jeton-A-factice", "jeton-B-factice"
	v := NewVerifier(a)
	steps := []struct {
		name   string
		tokens []string
		okA    bool
		okB    bool
	}{
		{"seul A", []string{a}, true, false},
		{"A et B (transition)", []string{a, b}, true, true},
		{"seul B", []string{b}, false, true},
	}
	for _, s := range steps {
		v.Set(s.tokens...)
		if v.Check(a) != s.okA || v.Check(b) != s.okB || v.Check("autre") {
			t.Fatalf("%s : A=%v B=%v", s.name, v.Check(a), v.Check(b))
		}
	}
}

var sinkB bool

func BenchmarkCheckWhich(b *testing.B) {
	v := NewVerifier("jeton-A-factice-aaaaaaaa", "jeton-B-factice-bbbbbbbb")
	for _, tc := range []struct{ name, tok string }{{"A", "jeton-A-factice-aaaaaaaa"}, {"B", "jeton-B-factice-bbbbbbbb"}, {"aucun", "xxxxxxxxxxxxxxxxxxxxxxxx"}} {
		b.Run(tc.name, func(b *testing.B) {
			for b.Loop() {
				sinkB = v.Check(tc.tok)
			}
		})
	}
}

// E4 : un « jeton » tiré de math/rand, initialisé par l'heure en secondes.
const alphabet = "abcdefghijklmnopqrstuvwxyz234567"

func weakToken(r *rand.Rand) string {
	b := make([]byte, 16)
	for i := range b {
		b[i] = alphabet[r.IntN(len(alphabet))]
	}
	return string(b)
}

func TestE4Predictable(t *testing.T) {
	created := time.Date(2026, 3, 1, 10, 0, 37, 0, time.UTC) // l'attaquant ne le sait qu'à la minute près
	gen := rand.New(rand.NewPCG(uint64(created.Unix()), 0))
	observed := weakToken(gen) // le jeton que l'attaquant a vu (le sien, par exemple)
	next := weakToken(gen)     // le jeton du client suivant

	start := time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC)
	tries := 0
	for s := start.Unix(); s < start.Unix()+60; s++ {
		tries++
		r := rand.New(rand.NewPCG(uint64(s), 0))
		if weakToken(r) == observed {
			guess := weakToken(r)
			fmt.Printf("graine retrouvée en %d essais ; jeton suivant deviné : %v\n", tries, guess == next)
			return
		}
	}
	t.Fatal("graine non retrouvée")
}
```

```
$ go test -v ./internal/auth/
--- PASS: TestRotation (0.00s)
graine retrouvée en 38 essais ; jeton suivant deviné : true
--- PASS: TestE4Predictable (0.00s)
$ go test -run XXX -bench CheckWhich -count 5 ./internal/auth/      # médianes
BenchmarkCheckWhich/A-8       268 ns/op
BenchmarkCheckWhich/B-8       275 ns/op
BenchmarkCheckWhich/aucun-8   266 ns/op
```

- **`Check` compare avec tous les jetons**, et combine les résultats par un « ou » binaire (`ok |= …`), sans
  `return true` anticipé. Un retour au premier jeton reconnu ferait répondre plus vite pour le premier que
  pour le second. Le temps ne dit pas **lequel** a été reconnu : 268, 275 et 266 ns, dans le bruit.
- **Le jeu de jetons** est remplacé d'un bloc par un `atomic.Pointer` (leçon 5) : on peut lancer la rotation
  pendant que des requêtes sont vérifiées.
- **La rotation** se fait en trois temps : on ajoute B, les clients passent à B, on retire A. Le test vérifie
  les trois états.

Le fichier remplace celui du cours (`NewVerifier` accepte maintenant plusieurs jetons), dont les tests
passent toujours.

## E4 — Pourquoi pas `math/rand` ?

Le test `TestE4Predictable` ci-dessus. Le générateur est initialisé par l'heure de création **en secondes**.
Connaissant cette heure à la minute près, l'attaquant essaie les 60 graines possibles, et retrouve la bonne
en **38 essais**, c'est-à-dire la seconde 37. Il produit alors avec la même graine le jeton **suivant**,
celui d'un autre client. Il a deviné juste.

Le jeton a l'air aléatoire (16 caractères parmi 32, soit 80 bits en apparence), mais son entropie réelle est
celle de la graine : **6 bits** ici (60 possibilités). `math/rand` produit une suite **déterminée** par sa
graine, ce qui est voulu pour une simulation et fatal pour un secret. `crypto/rand` puise dans le générateur
du système, qui n'a pas de graine devinable.

## E5 — Les modules privés

Sans réglage (sortie de `GOFLAGS=-x go get git.exemple.interne/equipe/lib@latest`) :

```
# get https://proxy.golang.org/git.exemple.interne/equipe/@v/list
# get https://proxy.golang.org/git.exemple.interne/@v/list
# get https://proxy.golang.org/git.exemple.interne/equipe/lib/@v/list
# get https://proxy.golang.org/git.exemple.interne/@v/list: 404 Not Found (1.533s)
[…]
```

Avec `GOPRIVATE=git.exemple.interne` :

```
# get https://git.exemple.interne/?go-get=1
# get https://git.exemple.interne/equipe/lib?go-get=1
# get https://git.exemple.interne/?go-get=1: […] lookup git.exemple.interne on 127.0.0.53:53: no such host
```

- **Sans `GOPRIVATE`**, le chemin complet du module, nom de l'hôte interne, de l'équipe et du projet
  compris, est envoyé au proxy **public**. Le proxy répond 404, puisqu'il ne peut pas y accéder, mais il a
  reçu la requête : c'est une fuite d'information sur l'organisation interne. Il en va de même pour
  `sum.golang.org` au moment de vérifier la somme.
- **Avec `GOPRIVATE`**, `go` contacte directement l'hôte. Ici, il n'existe pas, d'où l'erreur DNS. En vrai, il
  téléchargerait par Git, avec les identifiants de la machine, et ne consulterait pas la base publique de
  sommes. `GOPRIVATE` règle à la fois `GONOPROXY` et `GONOSUMDB`.

---

## Exercice intermédiaire — La revue de sécurité du service de prêts

La partie générique, les **tests de régression de sécurité**, sur le serveur durci d'un service :

```go
// Fichier : internal/server/server.go
// Package server : les deux serveurs d'un service, durcis (N7 L9, N10 L10, leçon 7).
package server

import (
	"net/http"
	"net/http/pprof"
	"time"
)

const maxBody = 1 << 20 // 1 Mio

// Public : l'API, et rien d'autre.
func Public(api http.Handler) *http.Server {
	return &http.Server{
		Handler:           secureHeaders(http.MaxBytesHandler(api, maxBody)),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
		MaxHeaderBytes:    16 << 10,
	}
}

// Admin : profils et métriques, sur une adresse séparée (127.0.0.1:6060).
func Admin(metrics http.Handler) *http.Server {
	mux := http.NewServeMux()
	mux.HandleFunc("/debug/pprof/", pprof.Index)
	mux.Handle("GET /metrics", metrics)
	return &http.Server{Handler: mux, ReadHeaderTimeout: 5 * time.Second}
}

func secureHeaders(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := w.Header()
		h.Set("X-Content-Type-Options", "nosniff")
		h.Set("X-Frame-Options", "DENY")
		h.Set("Content-Security-Policy", "default-src 'none'")
		h.Set("Referrer-Policy", "no-referrer")
		next.ServeHTTP(w, r)
	})
}
```

```go
// Fichier : internal/server/server_test.go
package server

import (
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func api() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /loans", func(w http.ResponseWriter, r *http.Request) {
		if _, err := io.ReadAll(r.Body); err != nil {
			http.Error(w, "corps trop grand", http.StatusRequestEntityTooLarge)
			return
		}
		w.WriteHeader(http.StatusCreated)
	})
	return mux
}

// Ces tests échouent si un point de la liste de durcissement régresse.
func TestPublicDoesNotExposeAdmin(t *testing.T) {
	srv := httptest.NewServer(Public(api()).Handler)
	defer srv.Close()
	for _, p := range []string{"/debug/pprof/", "/metrics", "/debug/pprof/goroutine?debug=2"} {
		resp, err := http.Get(srv.URL + p)
		if err != nil {
			t.Fatal(err)
		}
		resp.Body.Close()
		if resp.StatusCode != http.StatusNotFound && resp.StatusCode != http.StatusMethodNotAllowed {
			t.Errorf("%s : %d sur l'adresse publique", p, resp.StatusCode)
		}
	}
}

func TestTimeoutsAndLimits(t *testing.T) {
	s := Public(api())
	if s.ReadHeaderTimeout == 0 || s.ReadTimeout == 0 || s.WriteTimeout == 0 || s.IdleTimeout == 0 {
		t.Errorf("délais : %+v", s)
	}
	if s.MaxHeaderBytes == 0 || s.MaxHeaderBytes > 64<<10 {
		t.Errorf("MaxHeaderBytes = %d", s.MaxHeaderBytes)
	}
	if Admin(http.NotFoundHandler()).ReadHeaderTimeout == 0 {
		t.Error("le serveur d'administration n'a pas de délai")
	}
}

func TestSecurityHeaders(t *testing.T) {
	rec := httptest.NewRecorder()
	Public(api()).Handler.ServeHTTP(rec, httptest.NewRequest("POST", "/loans", strings.NewReader("{}")))
	for _, h := range []string{"X-Content-Type-Options", "X-Frame-Options", "Content-Security-Policy", "Referrer-Policy"} {
		if rec.Header().Get(h) == "" {
			t.Errorf("en-tête absent : %s", h)
		}
	}
}

func TestBodyTooLarge(t *testing.T) {
	rec := httptest.NewRecorder()
	big := strings.NewReader(strings.Repeat("x", maxBody+1))
	Public(api()).Handler.ServeHTTP(rec, httptest.NewRequest("POST", "/loans", big))
	if rec.Code != http.StatusRequestEntityTooLarge {
		t.Errorf("corps de %d octets : %d", maxBody+1, rec.Code)
	}
}
```

```
$ go test -v ./internal/server/
--- PASS: TestPublicDoesNotExposeAdmin (0.00s)
--- PASS: TestTimeoutsAndLimits (0.00s)
--- PASS: TestSecurityHeaders (0.00s)
--- PASS: TestBodyTooLarge (0.00s)
```

- **Chaque point de la liste** qui peut l'être devient un test : il ne régressera pas sans que la CI échoue.
  Les autres points se vérifient ailleurs : dans le Dockerfile (utilisateur, image), la CI (permissions),
  la configuration (secrets).
- **« Refuser un secret passé par option »** : avec la configuration de la leçon 5, il n'existe **aucune**
  option pour un secret. `-database-url=…` donne `flag provided but not defined`, et le service ne démarre
  pas (code 2). Le test consiste à vérifier qu'aucun nom de secret n'apparaît dans les options déclarées.
- **4. Les dépendances directes** : pour un service comme celui des niveaux 8 et 10, typiquement le pilote
  PostgreSQL (`pgx`), `golang.org/x/sync` (`errgroup`), éventuellement `golang.org/x/crypto` (mots de passe),
  Prometheus et OpenTelemetry (leçon 7). Chacune répond à un besoin que la bibliothèque standard ne couvre
  pas. Un module ajouté pour une fonction de dix lignes (un `slices.Contains`, un générateur d'UUID) est un
  candidat à la suppression.

---

## Défi — Signer une publication

**1 et 2.** Avec la publication de la leçon 2 :

```
$ ssh-keygen -q -t ed25519 -N '' -C 'publication svc' -f cle-publication
$ ssh-keygen -Y sign -f cle-publication -n fichier SHA256SUMS
Write signature to SHA256SUMS.sig
$ printf 'publication@exemple.org %s\n' "$(cut -d' ' -f1,2 cle-publication.pub)" > allowed_signers
$ ssh-keygen -Y verify -f allowed_signers -I publication@exemple.org -n fichier -s SHA256SUMS.sig < SHA256SUMS
Good "fichier" signature for publication@exemple.org with ED25519 key SHA256:mo7QGkFW2ORAgpGKDX3iBqeZ33Q+Hlj4zkj0DGivWeQ
$ sha256sum -c SHA256SUMS
svc-v1.2.0-linux-amd64: Réussi
svc-v1.2.0-linux-arm64: Réussi
svc-v1.2.0-windows-amd64.exe: Réussi
```

| Ce qui est modifié | Signature de `SHA256SUMS` | Sommes |
|---|---|---|
| un octet d'un binaire | valide | `svc-v1.2.0-linux-arm64: Échec` |
| un octet de `SHA256SUMS` | `Signature verification failed: incorrect signature` | (sans objet : on s'arrête là) |

Les deux vérifications se complètent. La signature dit que le fichier de sommes vient bien du détenteur de
la clé. Les sommes disent que les binaires sont ceux que ce fichier décrit. Un attaquant qui remplace un
binaire **et** recalcule `SHA256SUMS` est arrêté par la signature.

`-n fichier` est l'**espace de noms** de la signature : une signature faite pour `fichier` n'est pas acceptée
pour un autre usage, par exemple une signature de commit Git.

**3. Le vérificateur en Go :**

```go
// Fichier : cmd/verifyrelease/main.go
// Commande verifyrelease : vérifie une publication téléchargée (signature, puis sommes).
//
//	verifyrelease -dir dist -signers allowed_signers -identity publication@exemple.org
package main

import (
	"bufio"
	"bytes"
	"context"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"flag"
	"fmt"
	"io"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
)

func main() {
	dir := flag.String("dir", ".", "répertoire de la publication")
	signers := flag.String("signers", "allowed_signers", "clés autorisées (format de ssh-keygen)")
	identity := flag.String("identity", "", "identité attendue du signataire")
	flag.Parse()
	if err := Verify(context.Background(), *dir, *signers, *identity, os.Stdout); err != nil {
		fmt.Fprintln(os.Stderr, "ÉCHEC :", err)
		os.Exit(1)
	}
}

// Verify vérifie d'abord la signature de SHA256SUMS, PUIS chaque somme : une somme ne vaut
// rien tant qu'on ne sait pas qui a écrit le fichier de sommes.
func Verify(ctx context.Context, dir, signers, identity string, out io.Writer) error {
	if identity == "" {
		return errors.New("identité du signataire obligatoire")
	}
	sums, err := os.ReadFile(filepath.Join(dir, "SHA256SUMS"))
	if err != nil {
		return err
	}
	cmd := exec.CommandContext(ctx, "ssh-keygen", "-Y", "verify", "-f", signers, "-I", identity,
		"-n", "fichier", "-s", filepath.Join(dir, "SHA256SUMS.sig"))
	cmd.Stdin = bytes.NewReader(sums)
	if b, err := cmd.CombinedOutput(); err != nil {
		return fmt.Errorf("signature de SHA256SUMS : %w : %s", err, strings.TrimSpace(string(b)))
	}
	fmt.Fprintln(out, "signature de SHA256SUMS : valide")
	sc := bufio.NewScanner(bytes.NewReader(sums))
	n := 0
	for sc.Scan() {
		want, name, ok := strings.Cut(sc.Text(), "  ")
		if !ok || strings.ContainsAny(name, `/\`) { // pas de chemin : on ne lit que dans dir
			return fmt.Errorf("ligne invalide : %q", sc.Text())
		}
		data, err := os.ReadFile(filepath.Join(dir, name))
		if err != nil {
			return err
		}
		got := sha256.Sum256(data)
		if hex.EncodeToString(got[:]) != want {
			return fmt.Errorf("%s : somme différente", name)
		}
		fmt.Fprintf(out, "%s : ok\n", name)
		n++
	}
	if n == 0 {
		return errors.New("SHA256SUMS ne liste aucun fichier")
	}
	return sc.Err()
}
```

```go
// Fichier : cmd/verifyrelease/main_test.go
package main

import (
	"context"
	"crypto/sha256"
	"fmt"
	"io"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"testing"
)

// release construit une publication signée dans un répertoire temporaire.
func release(t *testing.T) (dir, signers string) {
	t.Helper()
	dir = t.TempDir()
	key := filepath.Join(t.TempDir(), "cle")
	run := func(name string, args ...string) {
		t.Helper()
		cmd := exec.Command(name, args...)
		cmd.Dir = dir
		if b, err := cmd.CombinedOutput(); err != nil {
			t.Fatalf("%s %v : %v\n%s", name, args, err, b)
		}
	}
	run("ssh-keygen", "-q", "-t", "ed25519", "-N", "", "-f", key)
	var sums strings.Builder
	for _, f := range []string{"svc-linux-amd64", "svc-linux-arm64"} {
		data := []byte("binaire " + f)
		os.WriteFile(filepath.Join(dir, f), data, 0o644)
		fmt.Fprintf(&sums, "%x  %s\n", sha256.Sum256(data), f)
	}
	os.WriteFile(filepath.Join(dir, "SHA256SUMS"), []byte(sums.String()), 0o644)
	run("ssh-keygen", "-Y", "sign", "-f", key, "-n", "fichier", "SHA256SUMS")
	pub, _ := os.ReadFile(key + ".pub")
	signers = filepath.Join(t.TempDir(), "allowed_signers")
	fields := strings.Fields(string(pub))
	os.WriteFile(signers, []byte("pub@exemple.org "+fields[0]+" "+fields[1]+"\n"), 0o644)
	return dir, signers
}

func TestVerify(t *testing.T) {
	if _, err := exec.LookPath("ssh-keygen"); err != nil {
		t.Skip("ssh-keygen absent")
	}
	ctx := context.Background()
	for _, tc := range []struct {
		name   string
		tamper func(dir string)
		id     string
		want   string
	}{
		{"intacte", func(string) {}, "pub@exemple.org", ""},
		{"binaire modifié", func(d string) { os.WriteFile(filepath.Join(d, "svc-linux-arm64"), []byte("piégé"), 0o644) }, "pub@exemple.org", "somme différente"},
		{"sommes modifiées", func(d string) {
			b, _ := os.ReadFile(filepath.Join(d, "SHA256SUMS"))
			os.WriteFile(filepath.Join(d, "SHA256SUMS"), append([]byte("0"), b[1:]...), 0o644)
		}, "pub@exemple.org", "signature"},
		{"autre signataire", func(string) {}, "intrus@exemple.org", "signature"},
	} {
		t.Run(tc.name, func(t *testing.T) {
			dir, signers := release(t)
			tc.tamper(dir)
			err := Verify(ctx, dir, signers, tc.id, io.Discard)
			if tc.want == "" && err != nil || tc.want != "" && (err == nil || !strings.Contains(err.Error(), tc.want)) {
				t.Fatalf("err = %v, attendu %q", err, tc.want)
			}
		})
	}
}
```

```
$ go test -v ./cmd/verifyrelease/
--- PASS: TestVerify (0.08s)
    --- PASS: TestVerify/intacte (0.02s)
    --- PASS: TestVerify/binaire_modifié (0.02s)
    --- PASS: TestVerify/sommes_modifiées (0.02s)
    --- PASS: TestVerify/autre_signataire (0.02s)
$ verifyrelease -dir dist -signers allowed_signers -identity publication@exemple.org
signature de SHA256SUMS : valide
svc-v1.2.0-linux-amd64 : ok
svc-v1.2.0-linux-arm64 : ok
svc-v1.2.0-windows-amd64.exe : ok
```

- **L'ordre compte** : la signature d'abord. Tant que le fichier de sommes n'est pas authentifié, ses
  sommes ne prouvent rien.
- **Un nom de fichier qui contient `/` est refusé** : un `SHA256SUMS` piégé ne doit pas faire lire
  `../../etc/passwd`.
- **Le test fabrique une vraie publication signée** dans `t.TempDir()`, avec une clé jetable, puis la
  modifie de quatre façons.

**4. La clé privée** ne vit **pas** sur un poste de développement ni dans le dépôt. Elle est dans un
gestionnaire de secrets, utilisable par la CI seulement, ou dans un module matériel. Si elle fuit, il faut :
- la révoquer, en la retirant d'`allowed_signers` et en publiant la nouvelle clé publique ;
- re-signer les publications avec la nouvelle clé ;
- prévenir les utilisateurs, car toute signature faite avec l'ancienne clé devient douteuse.

La signature « sans clé » (Sigstore, `cosign`) **supprime la clé longue durée**. Le job de CI obtient de
GitHub une identité temporaire (un jeton OIDC, `permissions: id-token: write`). Un service de Sigstore lui
délivre un certificat valable quelques minutes pour cette identité, il signe, et la signature est inscrite
dans un journal public. L'utilisateur vérifie non pas « signé par telle clé », mais « signé par **ce**
workflow, de **ce** dépôt, sur **cette** étiquette ». Il n'y a plus de clé à protéger ni à faire tourner. En
contrepartie, on dépend de l'infrastructure de Sigstore et de l'identité fournie par la CI. Cette variante
n'a pas été exécutée pour ce corrigé : elle demande une vraie CI.

---

## Quiz

1. Elle dit si le code **appelle** la fonction vulnérable (avec la trace), et pas seulement si le module est
   présent : on corrige d'abord ce qui est atteignable, sans noyer l'équipe sous des alertes théoriques.
2. Au moins une vulnérabilité est atteignable depuis le code analysé. En CI, l'étape échoue.
3. À analyser un binaire déjà construit ou publié, sans le code source, grâce aux informations de
   construction et aux symboles qu'il contient (leçon 2) : faut-il republier après une nouvelle alerte ?
4. Que tout le monde reçoit **le même** contenu pour une version donnée, et qu'il n'a pas changé depuis sa
   première publication. Pas que ce contenu est honnête, ni que le nom du module est le bon.
5. À désigner les modules privés : ni le proxy public ni la base de sommes ne sont interrogés pour eux. Leurs
   noms ne fuient pas, et ils sont téléchargés directement.
6. Création (aléatoire, `crypto/rand`), stockage (gestionnaire de secrets), distribution (fichier monté),
   usage (jamais journalisé), rotation (deux valeurs valides pendant la transition), révocation.
7. Le service accepte **deux** valeurs pendant la transition : on ajoute la nouvelle, les clients passent à
   elle, on retire l'ancienne. Il faut un rechargement à chaud ou un redémarrage sans interruption.
8. Le révoquer **immédiatement** et en créer un nouveau. Il reste dans l'historique (`git log -p -S`) et dans
   tous les clones. Nettoyer l'historique ensuite si besoin, et ajouter un analyseur de secrets.
9. `==` s'arrête au premier octet différent : son temps révèle combien d'octets sont justes (3,9 ns contre
   65 ns). Sur des empreintes, les valeurs comparées ont toujours la même longueur, et seule l'empreinte du
   secret est gardée en mémoire.
10. Le fichier de sommes prouve qu'un binaire correspond à ce fichier ; la signature prouve **qui** a écrit ce
    fichier. Sans elle, un attaquant qui remplace un binaire peut aussi remplacer les sommes.
