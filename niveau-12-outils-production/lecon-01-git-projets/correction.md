# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les scripts des exercices faciles partent du script `demo.sh` du cours, arrêté avant l'étape 6 (enregistré
sous `base.sh`). Les empreintes de commit sont donc les mêmes que dans le cours.

## E1 — Le binaire qui salit

```bash
source ./base.sh > /dev/null
echo "== E1"
rm -rf bin
sed -i '\|^/bin/$|d' .gitignore
git commit -qam "build: ne plus ignorer bin/"
GIT_COMMITTER_DATE="2026-03-03T10:00:00Z" git commit -q --amend --no-edit --date "2026-03-03T10:00:00Z"
go build -o app ./cmd/app && ./app
go build -o app ./cmd/app && ./app
git status --short
```

```
== E1
version v1.1.0-rc.1.0.20260303100000-83b638412e73, commit 83b638412e73, modifié false
version v1.1.0-rc.1.0.20260303100000-83b638412e73+dirty, commit 83b638412e73, modifié true
?? app
```

**Le premier build est propre** : au moment où `go build` interroge Git, le fichier `app` n'existe pas
encore. **Le second ne l'est pas** : le binaire produit par le premier est maintenant un fichier non suivi
(`?? app`), et `go build` le compte comme une modification.

Avec un binaire construit à la racine et non ignoré, toutes les reconstructions sont donc `+dirty`. Dans une
CI qui construit deux fois (tests, puis publication), le binaire publié porterait `+dirty`.

Au passage, une pseudo-version après une **préversion** a une forme différente :
`v1.1.0-rc.1.0.20260303100000-…`. C'est une version postérieure à `v1.1.0-rc.1`, et toujours antérieure à
`v1.1.0`.

## E2 — Lire un binaire de l'extérieur

```
$ go version -m app
app: go1.27.3
	path	example.com/app/cmd/app
	mod	example.com/app	v1.1.0-rc.1.0.20260303100000-83b638412e73+dirty
	build	-buildmode=exe
	build	-compiler=gc
	build	CGO_ENABLED=1
	…
	build	GOARCH=amd64
	build	GOOS=linux
	build	GOAMD64=v1
	build	vcs=git
	build	vcs.revision=83b638412e73805114c91063e01b6230942c05e1
	build	vcs.time=2026-03-03T10:00:00Z
	build	vcs.modified=true
```

Ce sont les lignes `mod` (version), `vcs.revision`, `vcs.time` et `vcs.modified`. Le binaire porte aussi
la version de Go, la plateforme et les options de construction : tout ce qu'il faut pour savoir quel code
tourne et comment il a été construit.

```
$ go version -m ~/go/bin/staticcheck
~/go/bin/staticcheck: go1.27.3
	path	honnef.co/go/tools/cmd/staticcheck
	mod	honnef.co/go/tools	v0.8.2	h1:+jPbCzuYxYrcoOvCecYUFDj+XJQRmzIhJmlbpps21Zk=
	dep	github.com/BurntSushi/toml	v1.4.1-0.20240526193622-a339e1f7089c	h1:…
	dep	golang.org/x/exp/typeparams	v0.0.0-20231108232855-2478ac86f678	h1:…
	dep	golang.org/x/mod	v0.41.0	h1:…
	dep	golang.org/x/sync	v0.23.0	h1:…
	dep	golang.org/x/tools	v0.51.0	h1:…
	…
```

Un outil installé par `go install` porte sa version et **toutes ses dépendances**, avec leurs sommes de
contrôle (`h1:…`). C'est ce que lit `govulncheck` (leçon 8) pour trouver les vulnérabilités d'un binaire
**sans son code source**.

On y voit aussi deux pseudo-versions de dépendances : `toml` et `x/exp/typeparams` sont pris à un commit
précis, et non à une version publiée.

## E3 — Ordonner des versions

Le paquet `golang.org/x/mod/semver` applique les règles de comparaison que l'outil `go` utilise lui-même. Le module des exercices le déclare :

```go.mod
// Fichier : go.mod
module exemple

go 1.27

require golang.org/x/mod v0.41.0
```

```go
// Fichier : semver_test.go
package exemple

import (
	"fmt"
	"testing"

	"golang.org/x/mod/semver"
)

func TestOrder(t *testing.T) {
	v := []string{"v1.2.0", "v1.10.0", "v1.2.1-0.20260101000000-abcdefabcdef", "v1.2.0-rc.2",
		"v1.2.0-rc.10", "v1.3.0-beta", "v0.9.9"}
	semver.Sort(v)
	for _, x := range v {
		fmt.Println(x)
	}
	if semver.Compare("v1.2.0-rc.10", "v1.2.0-rc.2") <= 0 {
		t.Fatal("rc.10 devrait suivre rc.2")
	}
}
```

```
v0.9.9
v1.2.0-rc.2
v1.2.0-rc.10
v1.2.0
v1.2.1-0.20260101000000-abcdefabcdef
v1.3.0-beta
v1.10.0
```

- **`v1.10.0` > `v1.3.0`** : on compare les nombres, pas les chaînes (en ordre alphabétique, `1.10` passerait
  avant `1.3`).
- **Une préversion précède sa version** : `v1.2.0-rc.2` < `v1.2.0`.
- **`rc.10` passe après `rc.2`** : dans une préversion, les parties séparées par des points se comparent une
  à une, et une partie **entièrement numérique** se compare comme un nombre. Une étiquette `rc10`, sans
  point, serait comparée comme du texte, et `rc10` passerait avant `rc2` : d'où la forme `-rc.N`.
- **La pseudo-version** `v1.2.1-0.…` est une préversion de `v1.2.1`. Elle se place après `v1.2.0` et avant
  `v1.3.0-beta`.

## E4 — Le tag déplacé

- **Ceux qui ont déjà téléchargé `v1.0.0`** gardent l'ancien code. Leur cache de modules et le proxy public
  (`proxy.golang.org`) en conservent une copie, que le proxy continue de servir.
- **Ceux qui passent par le proxy** reçoivent l'**ancien** code : le proxy ne relit pas l'étiquette.
- **Ceux qui téléchargent directement** depuis le dépôt (`GOPROXY=direct`, ou un module privé) reçoivent le
  **nouveau** code. Si ce module figure déjà dans leur `go.sum`, ou dans la base `sum.golang.org`, la
  vérification échoue avec `verifying module: checksum mismatch`.
- Deux personnes qui compilent « la même version » obtiennent donc deux programmes différents. Les builds
  ne sont plus reproductibles et les rapports de bug deviennent incompréhensibles.

À la place, on publie `v1.0.1` avec la correction, et on ajoute dans le `go.mod` de cette nouvelle version
`retract v1.0.0 // explication` (N2 L9). Les outils signalent alors la version retirée et ne la choisissent
plus d'eux-mêmes.

## E5 — Un `bisect` qui doit sauter

```bash
source ./base.sh > /dev/null
for i in 1 2 3 4 5; do
	case $i in
	2) echo "func cassé(" >> internal/price/price.go ;;                                # ne compile plus
	3) sed -i 's|cents \* 120 / 100|cents * 12 / 10 + 1|' internal/price/price.go ;; # la régression
	4) sed -i '/^func cassé($/d' internal/price/price.go ;;                            # recompile
	esac
	echo "// étape $i" >> cmd/app/main.go
	git commit -qam "refactor: étape $i"
	GIT_COMMITTER_DATE="2026-03-0$((2 + i))T10:00:00Z" git commit -q --amend --no-edit --date "2026-03-0$((2 + i))T10:00:00Z"
done
echo "== avec go test"
git bisect start HEAD v1.1.0-rc.1 > /dev/null
git bisect run go test ./internal/price/ > bisect.log 2>&1
grep -A5 "is the first bad commit" bisect.log | grep -E "first bad|    refactor"
git bisect reset > /dev/null 2>&1
cat > test.sh <<'SH'
#!/bin/sh
# 125 : « ce commit ne peut pas être testé », git bisect le saute.
go build ./... || exit 125
go test ./internal/price/
SH
chmod +x test.sh
echo "== avec test.sh"
git bisect start HEAD v1.1.0-rc.1 > /dev/null
git bisect run ./test.sh > bisect2.log 2>&1 || true
tail -6 bisect2.log
```

```
== avec go test
8b14212f87950bbf6d86e464a0324ac1e3a65cef is the first bad commit
    refactor: étape 2
== avec test.sh
The first bad commit could be any of:
4b7c9efb52a948439a70dcb8ef7aeded2f01f31a
8b14212f87950bbf6d86e464a0324ac1e3a65cef
38408fb73c956dea3d75bb1238f86fe9b1239e2c
Impossible de pousser la bissection plus loin !
error: la bissection ne peut plus continuer
```

(`8b14212` est l'étape 2, `4b7c9ef` l'étape 3, `38408fb` l'étape 4.)

- **Avec `go test`**, la réponse est **fausse**. L'étape 2 ne compile pas, `go test` échoue, et
  `bisect` la prend pour la régression. L'erreur est silencieuse : rien ne distingue « le test échoue » de
  « le code ne compile pas ».
- **Avec `test.sh`**, la réponse est **honnête**. Les étapes 2 et 3 ne compilent pas et sont sautées
  (code 125). La première étape testable qui échoue est l'étape 4. La régression peut donc se trouver dans
  l'étape 2, la 3 ou la 4, et `bisect` le dit. Il reste à examiner trois diffs au lieu de cent.

C'est la meilleure réponse possible avec un historique qui contient des commits qui ne compilent pas. D'où la
règle de la leçon : chaque commit compile et passe les tests.

---

## Exercice intermédiaire — Une commande `version`

```go
// Fichier : internal/buildinfo/buildinfo.go
// Package buildinfo expose ce que le binaire sait de sa construction.
package buildinfo

import (
	"encoding/json"
	"fmt"
	"net/http"
	"runtime"
	"runtime/debug"
	"time"
)

type Info struct {
	Version string    `json:"version"`
	Commit  string    `json:"commit"`
	Time    time.Time `json:"time,omitzero"`
	Dirty   bool      `json:"dirty"`
	Go      string    `json:"go"`
}

// Read lit les informations du binaire en cours d'exécution.
func Read() Info {
	bi, ok := debug.ReadBuildInfo()
	return From(bi, ok)
}

// From est la partie testable : elle ne dépend que de ses arguments.
func From(bi *debug.BuildInfo, ok bool) Info {
	info := Info{Version: "(inconnue)", Commit: "(inconnu)", Go: runtime.Version()}
	if !ok || bi == nil {
		return info
	}
	if bi.Main.Version != "" {
		info.Version = bi.Main.Version
	}
	for _, s := range bi.Settings {
		switch s.Key {
		case "vcs.revision":
			info.Commit = s.Value[:min(12, len(s.Value))]
		case "vcs.time":
			info.Time, _ = time.Parse(time.RFC3339, s.Value)
		case "vcs.modified":
			info.Dirty = s.Value == "true"
		}
	}
	return info
}

// Known indique si le binaire correspond à un commit identifiable et propre.
func (i Info) Known() bool {
	return i.Commit != "(inconnu)" && !i.Dirty
}

func (i Info) String() string {
	s := fmt.Sprintf("%s (commit %s, %s)", i.Version, i.Commit, i.Go)
	if !i.Known() {
		s += " — binaire non identifiable : ne pas déployer"
	}
	return s
}

// Handler répond à GET /version en JSON.
func Handler(i Info) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(i)
	})
}
```

```go
// Fichier : internal/buildinfo/buildinfo_test.go
package buildinfo

import (
	"encoding/json"
	"net/http/httptest"
	"runtime/debug"
	"testing"
	"time"
)

func TestFrom(t *testing.T) {
	full := &debug.BuildInfo{
		Main: debug.Module{Path: "exemple", Version: "v1.4.0"},
		Settings: []debug.BuildSetting{
			{Key: "vcs.revision", Value: "296a52ba0b8b4c1d2e3f40516273849506172839"},
			{Key: "vcs.time", Value: "2026-03-01T10:00:00Z"},
			{Key: "vcs.modified", Value: "false"},
		},
	}
	dirty := &debug.BuildInfo{
		Main:     debug.Module{Path: "exemple", Version: "v1.4.0+dirty"},
		Settings: []debug.BuildSetting{{Key: "vcs.revision", Value: "296a52ba0b8b"}, {Key: "vcs.modified", Value: "true"}},
	}
	devel := &debug.BuildInfo{Main: debug.Module{Path: "exemple", Version: "(devel)"}}
	for _, tc := range []struct {
		name    string
		bi      *debug.BuildInfo
		ok      bool
		version string
		commit  string
		known   bool
	}{
		{"complet", full, true, "v1.4.0", "296a52ba0b8b", true},
		{"modifié", dirty, true, "v1.4.0+dirty", "296a52ba0b8b", false},
		{"go run", devel, true, "(devel)", "(inconnu)", false},
		{"absent", nil, false, "(inconnue)", "(inconnu)", false},
	} {
		t.Run(tc.name, func(t *testing.T) {
			i := From(tc.bi, tc.ok)
			if i.Version != tc.version || i.Commit != tc.commit || i.Known() != tc.known {
				t.Fatalf("%+v", i)
			}
		})
	}
	if i := From(full, true); !i.Time.Equal(time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC)) {
		t.Fatalf("date : %v", i.Time)
	}
}

func TestHandler(t *testing.T) {
	rec := httptest.NewRecorder()
	Handler(Info{Version: "v1.4.0", Commit: "296a52ba0b8b", Go: "go1.27.3"}).ServeHTTP(rec, httptest.NewRequest("GET", "/version", nil))
	var got Info
	if err := json.NewDecoder(rec.Body).Decode(&got); err != nil {
		t.Fatal(err)
	}
	if got.Version != "v1.4.0" || rec.Header().Get("Content-Type") != "application/json" {
		t.Fatalf("%+v", got)
	}
}
```

```go
// Fichier : cmd/svc/main.go
// Commande svc : le démarrage d'un service, réduit à ce qui concerne sa version.
package main

import (
	"context"
	"flag"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"os/signal"

	"exemple/internal/buildinfo"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()
	if err := run(ctx, os.Args[1:], os.Stdout, buildinfo.Read()); err != nil {
		log.Fatal(err)
	}
}

func run(ctx context.Context, args []string, out io.Writer, info buildinfo.Info) error {
	fs := flag.NewFlagSet("svc", flag.ContinueOnError)
	fs.SetOutput(out)
	showVersion := fs.Bool("version", false, "affiche la version et quitte")
	admin := fs.String("admin", "127.0.0.1:6060", "adresse d'administration")
	if err := fs.Parse(args); err != nil {
		return err
	}
	if *showVersion {
		fmt.Fprintln(out, info)
		return nil
	}
	fmt.Fprintln(out, "démarrage :", info)
	mux := http.NewServeMux()
	mux.Handle("GET /version", buildinfo.Handler(info))
	srv := &http.Server{Addr: *admin, Handler: mux}
	go func() {
		<-ctx.Done()
		srv.Shutdown(context.Background())
	}()
	if err := srv.ListenAndServe(); err != http.ErrServerClosed {
		return err
	}
	return nil
}
```

```go
// Fichier : cmd/svc/main_test.go
package main

import (
	"bytes"
	"context"
	"testing"

	"exemple/internal/buildinfo"
)

func TestVersionFlag(t *testing.T) {
	var out bytes.Buffer
	info := buildinfo.Info{Version: "v1.4.0", Commit: "296a52ba0b8b", Go: "go1.27.3"}
	if err := run(context.Background(), []string{"-version"}, &out, info); err != nil {
		t.Fatal(err)
	}
	if got, want := out.String(), "v1.4.0 (commit 296a52ba0b8b, go1.27.3)\n"; got != want {
		t.Fatalf("%q, attendu %q", got, want)
	}
}
```

```
$ go run ./cmd/svc -version
(devel) (commit (inconnu), go1.27.3) — binaire non identifiable : ne pas déployer
```

- **3. Ce qu'on teste.** `debug.ReadBuildInfo` lit le binaire **en cours d'exécution**, et dans un test c'est
  le binaire de test, qui n'a pas de version. On sépare donc la lecture (`Read`, une ligne, non testée) de
  l'interprétation (`From`, une fonction pure, testée par une table). C'est l'injection de dépendances de
  N10 L3 à petite échelle.

  De même, `run` reçoit l'`Info` en paramètre : le test de `-version` lui passe une valeur fabriquée.
  Pour vérifier `Read` lui-même, il faudrait un test d'intégration qui construit le binaire dans un dépôt
  Git créé dans `t.TempDir()`, comme le défi le fait pour `git log`.
- **4.** `-version` est traité **avant** tout démarrage : pas de configuration requise, pas de port ouvert.
  `flag.ContinueOnError` et `fs.SetOutput(out)` rendent l'analyse des options testable : pas de
  `os.Exit`, et les messages vont dans `out`.
- **5. Informations absentes.** On ne remplace pas une valeur absente par une valeur plausible comme `v0.0.0`
  ou `dev`. On l'affiche telle quelle (`(devel)`, `(inconnu)`) et on avertit : le binaire n'est pas
  identifiable.

  On pourrait aller jusqu'à refuser de démarrer en production si `Known()` est faux. C'est une décision
  d'équipe. Le contrôle le plus sûr reste en CI, avec `-buildvcs=true` et un refus de publier un binaire
  `+dirty`.

---

## Défi — Des notes de version automatiques

```go
// Fichier : relnotes/relnotes.go
// Package relnotes produit des notes de version à partir des messages de commit.
package relnotes

import (
	"bytes"
	"context"
	"fmt"
	"os/exec"
	"regexp"
	"strconv"
	"strings"
)

// Commit est un message analysé.
type Commit struct {
	Hash     string
	Type     string // feat, fix, docs… ; vide si le message ne suit pas la convention
	Scope    string
	Summary  string
	Breaking bool
}

var header = regexp.MustCompile(`^([a-z]+)(?:\(([^)]+)\))?(!)?: (.+)$`)

// Parse analyse le titre et le corps d'un message.
func Parse(hash, subject, body string) Commit {
	c := Commit{Hash: hash, Summary: subject}
	m := header.FindStringSubmatch(subject)
	if m == nil {
		return c // hors convention
	}
	c.Type, c.Scope, c.Breaking, c.Summary = m[1], m[2], m[3] == "!", m[4]
	for line := range strings.Lines(body) {
		if strings.HasPrefix(line, "BREAKING CHANGE:") || strings.HasPrefix(line, "BREAKING-CHANGE:") {
			c.Breaking = true
		}
	}
	return c
}

// Log lit les commits entre from (exclu) et to, dans le dépôt dir.
func Log(ctx context.Context, dir, from, to string) ([]Commit, error) {
	// %x1f sépare les champs, %x1e les commits : ni l'un ni l'autre n'apparaît dans un message.
	cmd := exec.CommandContext(ctx, "git", "log", "--format=%H%x1f%s%x1f%b%x1e", from+".."+to)
	cmd.Dir = dir
	var stderr bytes.Buffer
	cmd.Stderr = &stderr
	out, err := cmd.Output()
	if err != nil {
		return nil, fmt.Errorf("git log : %w : %s", err, stderr.String())
	}
	var commits []Commit
	for rec := range strings.SplitSeq(string(out), "\x1e") {
		rec = strings.TrimSpace(rec)
		if rec == "" {
			continue
		}
		f := strings.SplitN(rec, "\x1f", 3)
		if len(f) != 3 {
			return nil, fmt.Errorf("enregistrement inattendu : %q", rec)
		}
		commits = append(commits, Parse(f[0], f[1], f[2]))
	}
	return commits, nil
}

// Next propose la version suivante de current (vMAJ.MIN.COR) d'après les commits.
func Next(current string, commits []Commit) (string, error) {
	var v [3]int
	parts := strings.Split(strings.TrimPrefix(current, "v"), ".")
	if len(parts) != 3 || !strings.HasPrefix(current, "v") {
		return "", fmt.Errorf("version invalide : %q", current)
	}
	for i, p := range parts {
		n, err := strconv.Atoi(p)
		if err != nil {
			return "", fmt.Errorf("version invalide : %q", current)
		}
		v[i] = n
	}
	breaking, feat := false, false
	for _, c := range commits {
		breaking = breaking || c.Breaking
		feat = feat || c.Type == "feat"
	}
	switch {
	case breaking && v[0] > 0:
		v = [3]int{v[0] + 1, 0, 0}
	case breaking || feat: // en v0, une rupture n'incrémente que la mineure
		v = [3]int{v[0], v[1] + 1, 0}
	default:
		v[2]++
	}
	return fmt.Sprintf("v%d.%d.%d", v[0], v[1], v[2]), nil
}

// Markdown écrit les notes, groupées. Les messages hors convention sont listés à part, pour
// qu'on les voie (et qu'on corrige l'habitude), sans faire échouer la publication.
func Markdown(version string, commits []Commit) string {
	var b strings.Builder
	fmt.Fprintf(&b, "## %s\n", version)
	section := func(title string, keep func(Commit) bool) {
		first := true
		for _, c := range commits {
			if !keep(c) {
				continue
			}
			if first {
				fmt.Fprintf(&b, "\n### %s\n\n", title)
				first = false
			}
			scope := ""
			if c.Scope != "" {
				scope = "**" + c.Scope + "** : "
			}
			fmt.Fprintf(&b, "- %s%s (%.7s)\n", scope, c.Summary, c.Hash)
		}
	}
	section("Ruptures de compatibilité", func(c Commit) bool { return c.Breaking })
	section("Fonctionnalités", func(c Commit) bool { return c.Type == "feat" && !c.Breaking })
	section("Corrections", func(c Commit) bool { return c.Type == "fix" && !c.Breaking })
	section("Messages hors convention", func(c Commit) bool { return c.Type == "" })
	return b.String()
}
```

```go
// Fichier : relnotes/relnotes_test.go
package relnotes

import (
	"context"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"testing"
)

func TestParse(t *testing.T) {
	for _, tc := range []struct {
		subject, body string
		want          Commit
	}{
		{"feat(api): pagination", "", Commit{Type: "feat", Scope: "api", Summary: "pagination"}},
		{"fix: arrondi des prix", "", Commit{Type: "fix", Summary: "arrondi des prix"}},
		{"feat!: nouveau format", "", Commit{Type: "feat", Summary: "nouveau format", Breaking: true}},
		{"refactor(db): pool", "Détails.\n\nBREAKING CHANGE: DSN obligatoire\n", Commit{Type: "refactor", Scope: "db", Summary: "pool", Breaking: true}},
		{"modifs diverses", "", Commit{Summary: "modifs diverses"}},
		{"Fix: majuscule", "", Commit{Summary: "Fix: majuscule"}},
	} {
		got := Parse("", tc.subject, tc.body)
		if got != tc.want {
			t.Errorf("%q : %+v, attendu %+v", tc.subject, got, tc.want)
		}
	}
}

func TestNext(t *testing.T) {
	feat, fix, brk := Commit{Type: "feat"}, Commit{Type: "fix"}, Commit{Type: "fix", Breaking: true}
	for _, tc := range []struct {
		cur     string
		commits []Commit
		want    string
	}{
		{"v1.4.2", []Commit{fix}, "v1.4.3"},
		{"v1.4.2", []Commit{fix, feat}, "v1.5.0"},
		{"v1.4.2", []Commit{feat, brk}, "v2.0.0"},
		{"v0.3.1", []Commit{brk}, "v0.4.0"},
		{"v0.3.1", nil, "v0.3.2"},
	} {
		if got, err := Next(tc.cur, tc.commits); err != nil || got != tc.want {
			t.Errorf("Next(%s) = %s, %v ; attendu %s", tc.cur, got, err, tc.want)
		}
	}
	if _, err := Next("1.2", nil); err == nil {
		t.Error("version invalide acceptée")
	}
}

// Un vrai dépôt, des dates fixées.
func TestLogRepo(t *testing.T) {
	dir := t.TempDir()
	git := func(env []string, args ...string) {
		t.Helper()
		cmd := exec.Command("git", args...)
		cmd.Dir = dir
		cmd.Env = append(os.Environ(), "GIT_AUTHOR_NAME=Dev", "GIT_AUTHOR_EMAIL=dev@example.com",
			"GIT_COMMITTER_NAME=Dev", "GIT_COMMITTER_EMAIL=dev@example.com")
		cmd.Env = append(cmd.Env, env...)
		if out, err := cmd.CombinedOutput(); err != nil {
			t.Fatalf("git %v : %v\n%s", args, err, out)
		}
	}
	commit := func(day int, msg string) {
		date := fmt.Sprintf("2026-03-%02dT10:00:00Z", day)
		os.WriteFile(filepath.Join(dir, "f.txt"), []byte(msg), 0o644)
		git(nil, "add", ".")
		git([]string{"GIT_AUTHOR_DATE=" + date, "GIT_COMMITTER_DATE=" + date}, "commit", "-q", "-m", msg)
	}
	git(nil, "init", "-q", "-b", "main")
	commit(1, "feat: première version")
	git([]string{"GIT_COMMITTER_DATE=2026-03-01T10:00:00Z"}, "tag", "-a", "v1.0.0", "-m", "v1.0.0")
	commit(2, "fix(prix): arrondi au centime")
	commit(3, "feat(api): pagination\n\nRefs: #12")
	commit(4, "mise à jour")

	commits, err := Log(context.Background(), dir, "v1.0.0", "HEAD")
	if err != nil {
		t.Fatal(err)
	}
	next, err := Next("v1.0.0", commits)
	if err != nil {
		t.Fatal(err)
	}
	fmt.Print(Markdown(next, commits))
	if next != "v1.1.0" || len(commits) != 3 {
		t.Fatalf("%s, %d commits", next, len(commits))
	}
}
```

```
$ go test -v ./relnotes/
--- PASS: TestParse (0.00s)
--- PASS: TestNext (0.00s)
## v1.1.0

### Fonctionnalités

- **api** : pagination (ea773a9)

### Corrections

- **prix** : arrondi au centime (b51d64b)

### Messages hors convention

- mise à jour (39ec5de)
--- PASS: TestLogRepo (0.04s)
```

- **Le découpage.** `%x1f` et `%x1e` sont les caractères ASCII « séparateur d'unité » et « séparateur
  d'enregistrement ». Ils n'apparaissent pas dans un message de commit, et le corps (`%b`), qui peut contenir
  des sauts de ligne, se découpe donc sans ambiguïté. Un format avec `|` ou des sauts de ligne casserait au
  premier message qui en contient.
- **`exec.CommandContext` sans shell.** Les arguments (`from+".."+to`) sont passés tels quels à `git`, sans
  interprétation. Une étiquette nommée `v1; rm -rf ~` reste un argument bizarre, pas une commande. La sortie
  d'erreur est capturée pour que l'erreur dise **pourquoi** `git` a échoué (étiquette inconnue, pas un
  dépôt).
- **Les versions v0.** En `v0.x.y`, l'API n'est pas stable par définition, et une rupture n'incrémente que la
  mineure (`v0.3.1` → `v0.4.0`). Passer en `v1.0.0` est une **décision** humaine, pas une conséquence d'un
  message de commit.
- **Le test avec un vrai dépôt.** Dates et auteur fixés : les empreintes (`ea773a9`…) sont les mêmes à chaque
  exécution, et la sortie pourrait servir de fichier de référence (*golden file*, N5 L4).
- **4. Les messages hors convention** sont listés **à part**, ni ignorés ni bloquants. Les ignorer ferait
  disparaître des changements des notes de version. Faire échouer la publication pour un message mal formé
  serait disproportionné : l'historique ne se réécrit pas une fois poussé.

  Le bon endroit pour **imposer** la convention est en amont. On vérifie le titre de la *pull request*
  en CI (leçon 4), ou celui des commits avant la fusion, quand ils peuvent encore être corrigés.

---

## Quiz

1. On valide `go.mod`, `go.sum`, le code (y compris le code généré si le build ne le régénère pas) et
   `testdata/`. On ignore les binaires, `*.test`, les profils et la couverture (`*.out`), et les secrets
   locaux (`.env`).
2. Une branche principale toujours déployable, et des branches courtes (heures, jours) fusionnées après revue
   et CI verte. Les fonctionnalités inachevées sont fusionnées désactivées, derrière un drapeau.
3. **Pourquoi** le changement est fait : le problème, la décision, les options écartées, et les références
   (ticket). Le **comment** est dans le diff.
4. `type!:` dans le titre (`feat!: …`), ou une ligne `BREAKING CHANGE: …` dans le corps.
5. Le proxy et la base de sommes de contrôle ont enregistré la version. Selon leur chemin, les utilisateurs
   verraient l'ancien ou le nouveau code, avec des erreurs `checksum mismatch`. Les builds ne seraient plus
   reproductibles. On publie une nouvelle version, et on retire l'ancienne avec `retract`.
6. Une pseudo-version `v1.4.3-0.AAAAMMJJhhmmss-empreinte12`, c'est-à-dire une préversion de `v1.4.3`.
7. Le dépôt contenait des modifications non validées au moment du build, fichiers non suivis compris. La
   cause classique est le binaire construit à la racine et absent du `.gitignore`.
8. `(devel)`. `go run` n'inscrit pas les informations Git.
9. Une recherche dichotomique : la commande est lancée sur des commits de l'intervalle, et son code de retour
   (0 = bon, autre = mauvais) réduit l'intervalle jusqu'au premier commit mauvais. Le code 125 saute un
   commit qui ne peut pas être testé.
10. Avec une étiquette préfixée par le chemin du sous-répertoire : `outils/v0.3.0` pour le module de
    `outils/`.
