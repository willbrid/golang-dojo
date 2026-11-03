# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Déplacer l'autorisation

Les tests du handler passent toujours (le 403 est produit, ailleurs). Mais `TestServiceAuthorization`
échoue : le service accepte désormais que `bob` emprunte pour `ada`. Et la fonction `ImportBatch`, qui
appelle le service directement, n'a **aucune** autorisation : quiconque peut écrire le fichier d'import
emprunte pour n'importe qui. Une règle d'autorisation placée dans un transport ne protège que ce
transport.

## E2 — Retirer la transaction

Avec un `WithinTx` qui travaille directement sur les maps (sans brouillons, mais toujours sous le
verrou), `TestServiceAtomicity` échoue : `SaveMember` a été appliqué, `SaveCopy` a échoué, le compteur
d'`ada` vaut 1 alors qu'elle n'a rien emprunté. Le test de concurrence passe encore, grâce au verrou.
Sans le verrou non plus, il **pourrait** passer par chance (deux goroutines qui ne se chevauchent pas), et
`-race` le signalerait ; c'est pourquoi il faut les deux tests, et `-race`.

## E3 — Un champ de trop

En encodant `Copy` directement : `{"ID":"c1","BorrowedBy":"ada","Due":"…"}`. Les noms de champs Go
deviennent l'API (et `BorrowedBy` révèle qui a emprunté, ce que l'appelant n'avait pas demandé). Avec
`InternalNote`, la note interne part chez **tous** les clients sans que personne n'ait rien décidé. Le
DTO `borrowResponse` choisit explicitement ce qui sort : un champ ajouté au domaine ne fuit pas.

## E4 — Le CRUD

Sans règle, une méthode de service `GetMember` ne ferait que transmettre au stockage : le handler peut
lire via le `Store` directement. Dès la règle « un adhérent ne voit que sa fiche, un bibliothécaire
toutes », c'est une **autorisation**, donc une décision du service : la méthode de service apparaît, et
le handler cesse de lire le stockage lui-même. Commencer sans, et l'ajouter au moment où la règle arrive,
est raisonnable — à condition de ne pas oublier de déplacer l'accès ce jour-là.

## E5 — Trois étages de validation

| Vérification | Étage | Statut |
|---|---|---|
| `copy_id` absent | handler (forme) | 400 |
| `copy_id` de 10 000 caractères | handler (`MaxBytesReader`, limite de taille) | 400 (ou 413) |
| exemplaire inexistant | stockage → erreur du domaine `ErrNotFound` | 404 |
| exemplaire déjà emprunté | domaine (`ErrUnavailable`) | 409 |
| deux emprunts simultanés (PostgreSQL) | base : verrou `FOR UPDATE` ou contrainte | 409 pour le perdant |
| corps qui n'est pas du JSON | handler | 400 |
| adhérent suspendu | domaine (`ErrSuspended`) | 409 |

## Exercice intermédiaire — Le retour, de bout en bout

Les fichiers du domaine, du service et du handler, complétés (le stockage en mémoire et les tests du
cours sont inchangés) :

```go
// Fichier : loan/domain.go
// Domaine : types et règles. Aucune notion de requête, d'utilisateur connecté ni de transaction.
package loan

import (
	"errors"
	"time"
)

var (
	ErrTooManyLoans = errors.New("nombre maximal d'emprunts atteint")
	ErrUnavailable  = errors.New("exemplaire déjà emprunté")
	ErrNotFound     = errors.New("introuvable")
	ErrForbidden    = errors.New("action non autorisée")
	ErrSuspended    = errors.New("adhérent suspendu")
	ErrNotBorrowed  = errors.New("exemplaire non emprunté")
)

const SuspensionThreshold = 7 // jours de retard au-delà desquels on suspend

type Member struct {
	ID             string
	Loans          int
	SuspendedUntil time.Time // zéro : pas de suspension
}

type Copy struct {
	ID         string
	BorrowedBy string // "" si disponible
	Due        time.Time
}

const MaxLoans = 3

// Lend : règle portée par les deux objets ; le service garantit qu'ils sont modifiés ensemble.
func Lend(m *Member, c *Copy, now time.Time) error {
	if c.BorrowedBy != "" {
		return ErrUnavailable
	}
	if now.Before(m.SuspendedUntil) {
		return ErrSuspended
	}
	if m.Loans >= MaxLoans {
		return ErrTooManyLoans
	}
	m.Loans++
	c.BorrowedBy, c.Due = m.ID, now.Add(21*24*time.Hour)
	return nil
}

// Receive : le retour. Rend les jours ENTAMÉS de retard ; suspend au-delà du seuil.
func Receive(m *Member, c *Copy, now time.Time) (int, error) {
	if c.BorrowedBy != m.ID {
		return 0, ErrNotBorrowed
	}
	late := 0
	if d := now.Sub(c.Due); d > 0 {
		late = int((d + 24*time.Hour - 1) / (24 * time.Hour))
	}
	if late > SuspensionThreshold {
		m.SuspendedUntil = now.Add(time.Duration(late) * 24 * time.Hour)
	}
	m.Loans--
	c.BorrowedBy, c.Due = "", time.Time{}
	return late, nil
}
```

```go
// Fichier : loan/service.go
package loan

import (
	"context"
	"fmt"
	"time"
)

// Repos : ce que le service attend du stockage, DANS une transaction.
type Repos interface {
	Member(ctx context.Context, id string) (Member, error)
	Copy(ctx context.Context, id string) (Copy, error)
	SaveMember(ctx context.Context, m Member) error
	SaveCopy(ctx context.Context, c Copy) error
}

// Store ouvre une transaction et donne les repositories qui y travaillent (N8 L7).
type Store interface {
	WithinTx(ctx context.Context, fn func(Repos) error) error
}

// Actor : QUI agit. Le handler l'établit (authentification) ; le service décide (autorisation).
type Actor struct {
	ID        string
	Librarian bool
}

type Service struct {
	store Store
	now   func() time.Time
}

func NewService(s Store, now func() time.Time) *Service { return &Service{store: s, now: now} }

// Borrow : un cas d'usage = une méthode = une transaction.
func (s *Service) Borrow(ctx context.Context, who Actor, memberID, copyID string) (Copy, error) {
	// Autorisation : une règle métier (« un adhérent n'emprunte que pour lui-même »).
	if who.ID != memberID && !who.Librarian {
		return Copy{}, fmt.Errorf("%s pour %s : %w", who.ID, memberID, ErrForbidden)
	}
	var lent Copy
	err := s.store.WithinTx(ctx, func(r Repos) error {
		m, err := r.Member(ctx, memberID)
		if err != nil {
			return err
		}
		c, err := r.Copy(ctx, copyID)
		if err != nil {
			return err
		}
		if err := Lend(&m, &c, s.now()); err != nil {
			return err
		}
		if err := r.SaveMember(ctx, m); err != nil {
			return err
		}
		lent = c
		return r.SaveCopy(ctx, c)
	})
	return lent, err
}

type ReturnResult struct {
	LateDays       int
	SuspendedUntil time.Time
}

func (s *Service) Return(ctx context.Context, who Actor, copyID string) (ReturnResult, error) {
	var res ReturnResult
	err := s.store.WithinTx(ctx, func(r Repos) error {
		c, err := r.Copy(ctx, copyID)
		if err != nil {
			return err
		}
		if c.BorrowedBy == "" {
			return fmt.Errorf("exemplaire %s : %w", copyID, ErrNotBorrowed)
		}
		// Autorisation : il faut d'abord lire l'exemplaire pour savoir qui l'a emprunté.
		if who.ID != c.BorrowedBy && !who.Librarian {
			return fmt.Errorf("%s rend %s : %w", who.ID, copyID, ErrForbidden)
		}
		m, err := r.Member(ctx, c.BorrowedBy)
		if err != nil {
			return err
		}
		if res.LateDays, err = Receive(&m, &c, s.now()); err != nil {
			return err
		}
		res.SuspendedUntil = m.SuspendedUntil
		if err := r.SaveMember(ctx, m); err != nil {
			return err
		}
		return r.SaveCopy(ctx, c)
	})
	return res, err
}
```

```go
// Fichier : loan/handler.go
package loan

import (
	"encoding/json"
	"errors"
	"net/http"
	"time"
)

// Les DTO : la forme de l'API, distincte des types du domaine (N7 L4, projet 3 D8).
type borrowRequest struct {
	CopyID string `json:"copy_id"`
}

type borrowResponse struct {
	CopyID string    `json:"copy_id"`
	Due    time.Time `json:"due"`
}

func Routes(svc *Service) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /members/{id}/loans", func(w http.ResponseWriter, r *http.Request) {
		// 1. Authentification (simulée ici) : établir QUI. Pas de décision métier.
		who := Actor{ID: r.Header.Get("X-User"), Librarian: r.Header.Get("X-Role") == "librarian"}
		if who.ID == "" {
			http.Error(w, "non authentifié", http.StatusUnauthorized)
			return
		}
		// 2. Validation SYNTAXIQUE : le corps est-il bien formé ?
		var in borrowRequest
		dec := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<10))
		dec.DisallowUnknownFields()
		if err := dec.Decode(&in); err != nil || in.CopyID == "" {
			http.Error(w, "corps invalide : {\"copy_id\": \"…\"} attendu", http.StatusBadRequest)
			return
		}
		// 3. Le cas d'usage.
		c, err := svc.Borrow(r.Context(), who, r.PathValue("id"), in.CopyID)
		if err != nil {
			http.Error(w, err.Error(), status(err))
			return
		}
		// 4. Domaine → DTO → JSON.
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(borrowResponse{CopyID: c.ID, Due: c.Due})
	})
	mux.HandleFunc("DELETE /loans/{copy}", func(w http.ResponseWriter, r *http.Request) {
		who := Actor{ID: r.Header.Get("X-User"), Librarian: r.Header.Get("X-Role") == "librarian"}
		if who.ID == "" {
			http.Error(w, "non authentifié", http.StatusUnauthorized)
			return
		}
		res, err := svc.Return(r.Context(), who, r.PathValue("copy"))
		if err != nil {
			http.Error(w, err.Error(), status(err))
			return
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(returnResponse{LateDays: res.LateDays, SuspendedUntil: res.SuspendedUntil})
	})
	return mux
}

type returnResponse struct {
	LateDays       int       `json:"late_days"`
	SuspendedUntil time.Time `json:"suspended_until,omitzero"` // absent si pas de suspension
}

func status(err error) int {
	switch {
	case errors.Is(err, ErrForbidden):
		return http.StatusForbidden
	case errors.Is(err, ErrNotFound):
		return http.StatusNotFound
	case errors.Is(err, ErrTooManyLoans), errors.Is(err, ErrUnavailable), errors.Is(err, ErrSuspended),
		errors.Is(err, ErrNotBorrowed):
		return http.StatusConflict
	default:
		return http.StatusInternalServerError
	}
}
```

```go
// Fichier : loan/return_test.go
package loan

import (
	"errors"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

func TestReceiveTable(t *testing.T) {
	due := now.Add(21 * 24 * time.Hour)
	for _, c := range []struct {
		at        time.Time
		late      int
		suspended bool
	}{
		{due, 0, false},
		{due.Add(time.Minute), 1, false},
		{due.Add(7 * 24 * time.Hour), 7, false},
		{due.Add(7*24*time.Hour + time.Second), 8, true},
	} {
		m, cp := Member{ID: "ada", Loans: 1}, Copy{ID: "c1", BorrowedBy: "ada", Due: due}
		late, err := Receive(&m, &cp, c.at)
		if err != nil || late != c.late || (!m.SuspendedUntil.IsZero()) != c.suspended || m.Loans != 0 || cp.BorrowedBy != "" {
			t.Errorf("à %v : late=%d susp=%v err=%v", c.at.Sub(due), late, m.SuspendedUntil, err)
		}
	}
}

func TestReturnService(t *testing.T) {
	svc, st := newSvc()
	ctx := t.Context()
	if _, err := svc.Borrow(ctx, Actor{ID: "ada"}, "ada", "c1"); err != nil {
		t.Fatal(err)
	}
	if _, err := svc.Return(ctx, Actor{ID: "bob"}, "c1"); !errors.Is(err, ErrForbidden) {
		t.Fatalf("bob rend le livre d'ada : %v", err)
	}
	st.FailOn = "c1"
	if _, err := svc.Return(ctx, Actor{ID: "ada"}, "c1"); err == nil {
		t.Fatal("panne attendue")
	}
	if m, _ := st.Snapshot(); m["ada"].Loans != 1 {
		t.Fatalf("retour partiellement appliqué : %+v", m["ada"])
	}
	st.FailOn = ""
	if _, err := svc.Return(ctx, Actor{ID: "lib", Librarian: true}, "c1"); err != nil {
		t.Fatal(err)
	}
	if _, err := svc.Return(ctx, Actor{ID: "ada"}, "c1"); !errors.Is(err, ErrNotBorrowed) {
		t.Fatalf("double retour : %v", err)
	}
}

func TestSuspendedCannotBorrow(t *testing.T) {
	cur := now
	st := NewMemStore([]Member{{ID: "ada"}}, []Copy{{ID: "c1"}, {ID: "c2"}})
	svc := NewService(st, func() time.Time { return cur })
	ctx := t.Context()
	svc.Borrow(ctx, Actor{ID: "ada"}, "ada", "c1")
	cur = cur.Add(31 * 24 * time.Hour) // 10 jours de retard
	res, err := svc.Return(ctx, Actor{ID: "ada"}, "c1")
	if err != nil || res.LateDays != 10 {
		t.Fatalf("%+v %v", res, err)
	}
	if _, err := svc.Borrow(ctx, Actor{ID: "ada"}, "ada", "c2"); !errors.Is(err, ErrSuspended) {
		t.Fatalf("emprunt pendant la suspension : %v", err)
	}
	cur = res.SuspendedUntil
	if _, err := svc.Borrow(ctx, Actor{ID: "ada"}, "ada", "c2"); err != nil {
		t.Fatalf("après la suspension : %v", err)
	}
}

func TestReturnHandler(t *testing.T) {
	cur := now
	st := NewMemStore([]Member{{ID: "ada"}}, []Copy{{ID: "c1"}})
	h := Routes(NewService(st, func() time.Time { return cur }))
	req := httptest.NewRequest("POST", "/members/ada/loans", strings.NewReader(`{"copy_id":"c1"}`))
	req.Header.Set("X-User", "ada")
	h.ServeHTTP(httptest.NewRecorder(), req)
	cur = cur.Add(22 * 24 * time.Hour)
	req = httptest.NewRequest("DELETE", "/loans/c1", nil)
	req.Header.Set("X-User", "ada")
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, req)
	if rec.Code != 200 || strings.TrimSpace(rec.Body.String()) != `{"late_days":1}` {
		t.Fatalf("%d %s", rec.Code, rec.Body)
	}
}
```

Où vivent les règles :

- **« seul l'emprunteur ou un bibliothécaire peut rendre »** : autorisation, donc service — mais elle ne
  peut être vérifiée **qu'après** avoir lu l'exemplaire (pour savoir qui l'a emprunté) : elle est donc à
  l'intérieur de la transaction, contrairement à celle de `Borrow`. Les deux sont correctes ; l'ordre
  découle des données nécessaires ;
- **calcul du retard et suspension** : domaine (`Receive`), testés par table aux limites (7 jours pile :
  pas de suspension ; 7 jours et une seconde : 8 jours entamés, suspension) ;
- **suspension qui empêche d'emprunter** : domaine (`Lend` vérifie `SuspendedUntil`).

`omitzero` (Go 1.24) omet `suspended_until` quand la date est zéro : `{"late_days":1}`.

**La couche service est-elle justifiée ?** Ici oui : elle porte deux autorisations différentes, la
transaction sur deux objets, et l'orchestration lecture → règle → écriture. Elle représente environ un
quart des lignes ; le domaine (les règles et leurs tests) le reste de la valeur.

## Défi — Un stockage PostgreSQL pour `Store`

```go
// Fichier : loan/storetest/contract.go
// Package storetest : le contrat de loan.Store, pour toute implémentation.
package storetest

import (
	"context"
	"errors"
	"sync"
	"testing"
	"time"

	"exemple/loan"
)

// New rend un Store vierge contenant les adhérents et exemplaires donnés.
type New func(t *testing.T, members []loan.Member, copies []loan.Copy) loan.Store

func Run(t *testing.T, newStore New) {
	t.Run("commit", func(t *testing.T) {
		st := newStore(t, []loan.Member{{ID: "ada"}}, []loan.Copy{{ID: "c1"}})
		svc := loan.NewService(st, time.Now)
		if _, err := svc.Borrow(t.Context(), loan.Actor{ID: "ada"}, "ada", "c1"); err != nil {
			t.Fatal(err)
		}
		check(t, st, "c1", "ada", 1)
	})
	t.Run("rollback", func(t *testing.T) {
		st := newStore(t, []loan.Member{{ID: "ada"}}, []loan.Copy{{ID: "c1"}})
		boom := errors.New("boum")
		err := st.WithinTx(t.Context(), func(r loan.Repos) error {
			m, _ := r.Member(t.Context(), "ada")
			m.Loans = 99
			if err := r.SaveMember(t.Context(), m); err != nil {
				return err
			}
			return boom
		})
		if !errors.Is(err, boom) {
			t.Fatalf("err = %v", err)
		}
		check(t, st, "c1", "", 0)
	})
	t.Run("introuvable", func(t *testing.T) {
		st := newStore(t, nil, nil)
		err := st.WithinTx(t.Context(), func(r loan.Repos) error {
			_, err := r.Copy(t.Context(), "zz")
			return err
		})
		if !errors.Is(err, loan.ErrNotFound) {
			t.Fatalf("err = %v", err)
		}
	})
	t.Run("concurrence", func(t *testing.T) {
		var members []loan.Member
		for i := range 50 {
			members = append(members, loan.Member{ID: string(rune('A' + i))})
		}
		st := newStore(t, members, []loan.Copy{{ID: "c1"}})
		svc := loan.NewService(st, time.Now)
		var wg sync.WaitGroup
		var mu sync.Mutex
		ok := 0
		for _, m := range members {
			wg.Go(func() {
				if _, err := svc.Borrow(t.Context(), loan.Actor{ID: m.ID}, m.ID, "c1"); err == nil {
					mu.Lock()
					ok++
					mu.Unlock()
				} else if !errors.Is(err, loan.ErrUnavailable) {
					t.Error(err)
				}
			})
		}
		wg.Wait()
		if ok != 1 {
			t.Fatalf("%d emprunts réussis du même exemplaire", ok)
		}
	})
}

func check(t *testing.T, st loan.Store, copyID, borrower string, loans int) {
	t.Helper()
	err := st.WithinTx(context.Background(), func(r loan.Repos) error {
		c, err := r.Copy(context.Background(), copyID)
		if err != nil {
			return err
		}
		if c.BorrowedBy != borrower {
			t.Errorf("exemplaire emprunté par %q, attendu %q", c.BorrowedBy, borrower)
		}
		if borrower == "" {
			borrower = "ada"
		}
		m, err := r.Member(context.Background(), borrower)
		if err != nil {
			return err
		}
		if m.Loans != loans {
			t.Errorf("%s : %d emprunts, attendu %d", borrower, m.Loans, loans)
		}
		return nil
	})
	if err != nil {
		t.Fatal(err)
	}
}
```

```go
// Fichier : loan/memstore_test.go
package loan_test

import (
	"testing"

	"exemple/loan"
	"exemple/loan/storetest"
)

func TestMemStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T, m []loan.Member, c []loan.Copy) loan.Store {
		return loan.NewMemStore(m, c)
	})
}
```

```go
// Fichier : loan/pgstore/pgstore.go
// Package pgstore : loan.Store avec PostgreSQL.
package pgstore

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"time"

	"exemple/loan"
)

const Schema = `
	CREATE TABLE IF NOT EXISTS members (
		id text PRIMARY KEY, loans int NOT NULL DEFAULT 0 CHECK (loans >= 0),
		suspended_until timestamptz);
	CREATE TABLE IF NOT EXISTS copies (
		id text PRIMARY KEY, borrowed_by text REFERENCES members(id), due timestamptz);`

type Store struct{ db *sql.DB }

func New(db *sql.DB) *Store { return &Store{db: db} }

type tx struct{ tx *sql.Tx }

func (s *Store) WithinTx(ctx context.Context, fn func(loan.Repos) error) error {
	t, err := s.db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer t.Rollback()
	if err := fn(&tx{t}); err != nil {
		return err
	}
	return t.Commit()
}

// Les lectures verrouillent (FOR UPDATE) : « lire puis écrire » sans mise à jour perdue (N8 L4).
// Ordre des verrous : le service lit toujours l'adhérent puis l'exemplaire (Borrow) ou
// l'exemplaire puis l'adhérent (Return) ; deux cas d'usage croisés peuvent s'interbloquer,
// PostgreSQL détecte et annule l'un (40P01) — à réessayer (N8 L4) ou à ordonner.
func (t *tx) Member(ctx context.Context, id string) (loan.Member, error) {
	var m loan.Member
	var susp sql.Null[time.Time]
	err := t.tx.QueryRowContext(ctx,
		`SELECT id, loans, suspended_until FROM members WHERE id = $1 FOR UPDATE`, id).Scan(&m.ID, &m.Loans, &susp)
	if errors.Is(err, sql.ErrNoRows) {
		return m, fmt.Errorf("adhérent %s : %w", id, loan.ErrNotFound)
	}
	m.SuspendedUntil = susp.V.UTC()
	if !susp.Valid {
		m.SuspendedUntil = time.Time{}
	}
	return m, err
}

func (t *tx) Copy(ctx context.Context, id string) (loan.Copy, error) {
	var c loan.Copy
	var by sql.Null[string]
	var due sql.Null[time.Time]
	err := t.tx.QueryRowContext(ctx,
		`SELECT id, borrowed_by, due FROM copies WHERE id = $1 FOR UPDATE`, id).Scan(&c.ID, &by, &due)
	if errors.Is(err, sql.ErrNoRows) {
		return c, fmt.Errorf("exemplaire %s : %w", id, loan.ErrNotFound)
	}
	c.BorrowedBy = by.V
	if due.Valid {
		c.Due = due.V.UTC()
	}
	return c, err
}

func (t *tx) SaveMember(ctx context.Context, m loan.Member) error {
	susp := sql.Null[time.Time]{V: m.SuspendedUntil, Valid: !m.SuspendedUntil.IsZero()}
	_, err := t.tx.ExecContext(ctx,
		`UPDATE members SET loans = $2, suspended_until = $3 WHERE id = $1`, m.ID, m.Loans, susp)
	return err
}

func (t *tx) SaveCopy(ctx context.Context, c loan.Copy) error {
	by := sql.Null[string]{V: c.BorrowedBy, Valid: c.BorrowedBy != ""}
	due := sql.Null[time.Time]{V: c.Due, Valid: !c.Due.IsZero()}
	_, err := t.tx.ExecContext(ctx, `UPDATE copies SET borrowed_by = $2, due = $3 WHERE id = $1`, c.ID, by, due)
	return err
}
```

```go
// Fichier : loan/pgstore/pgstore_test.go
package pgstore

import (
	"context"
	"crypto/rand"
	"database/sql"
	"os"
	"strings"
	"testing"

	_ "github.com/jackc/pgx/v5/stdlib"

	"exemple/loan"
	"exemple/loan/storetest"
)

func TestContract(t *testing.T) {
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	storetest.Run(t, func(t *testing.T, members []loan.Member, copies []loan.Copy) loan.Store {
		admin, err := sql.Open("pgx", dsn)
		if err != nil {
			t.Fatal(err)
		}
		t.Cleanup(func() { admin.Close() })
		schema := "test_" + strings.ToLower(rand.Text()[:12])
		if _, err := admin.ExecContext(t.Context(), "CREATE SCHEMA "+schema); err != nil {
			t.Fatal(err)
		}
		t.Cleanup(func() { admin.ExecContext(context.Background(), "DROP SCHEMA "+schema+" CASCADE") })
		db, err := sql.Open("pgx", dsn+"&search_path="+schema)
		if err != nil {
			t.Fatal(err)
		}
		db.SetMaxOpenConns(20)
		t.Cleanup(func() { db.Close() })
		if _, err := db.ExecContext(t.Context(), Schema); err != nil {
			t.Fatal(err)
		}
		for _, m := range members {
			db.ExecContext(t.Context(), `INSERT INTO members (id) VALUES ($1)`, m.ID)
		}
		for _, c := range copies {
			db.ExecContext(t.Context(), `INSERT INTO copies (id) VALUES ($1)`, c.ID)
		}
		return New(db)
	})
}
```

La suite de contrat passe contre `MemStore` et contre PostgreSQL (un schéma par test, N8 L7), y compris
50 emprunts simultanés du même exemplaire : un seul réussit sur chacune. Côté PostgreSQL, c'est le
`FOR UPDATE` qui sérialise les transactions : la deuxième attend, relit un exemplaire déjà emprunté, et le
domaine répond `ErrUnavailable`.

Remarque : `Borrow` verrouille l'adhérent puis l'exemplaire, `Return` l'exemplaire puis l'adhérent. Deux
cas d'usage croisés sur les mêmes objets peuvent donc s'interbloquer ; PostgreSQL en annule un (`40P01`).
Correctifs possibles : réessayer (N8 L4), ou imposer un ordre unique (toujours l'exemplaire d'abord).

**Point 4 — un verrou par exemplaire en mémoire.** Le test de concurrence passe (les emprunts d'un même
exemplaire sont sérialisés), mais le contrat devient faux ailleurs : deux emprunts d'exemplaires
**différents** par le **même adhérent** modifient tous deux son compteur, sans verrou commun, et peuvent
dépasser `MaxLoans` ou perdre un incrément. Il faut verrouiller **aussi** l'adhérent, et toujours dans le
même ordre (sinon interblocage) : c'est exactement ce que fait PostgreSQL avec ses verrous de ligne. Le
verrou global est plus lent mais trivialement correct — un bon choix pour un faux de test.

## Quiz

1. Il décode, valide la forme, établit l'acteur, appelle le service, traduit l'erreur, encode ; il ne
   décide d'aucune règle ni ne touche au stockage.
2. Syntaxique : la forme (handler) ; métier : les règles (domaine) ; intégrité : contraintes (base).
3. C'est une règle métier, qui doit s'appliquer quelle que soit la porte d'entrée.
4. Parce que c'est le cas d'usage qui définit ce qui doit être atomique ; via une interface `WithinTx`.
5. Un type qui définit la forme de l'API ; encoder le domaine fait fuir ses champs et lie l'API à sa
   représentation.
6. Pour que le service exprime ce que fait le métier, pas des opérations techniques.
7. Ce qui change ensemble est rangé ensemble, et les détails peuvent rester non exportés.
8. Quand il se contente de transmettre au repository sans règle, autorisation ni transaction.
9. Pour ne pas ouvrir de transaction (ni prendre de verrous) pour une demande refusée d'avance.
10. Le même contrat que la vraie implémentation : atomicité, isolation, mêmes erreurs — vérifié par une
    suite de contrat commune.
