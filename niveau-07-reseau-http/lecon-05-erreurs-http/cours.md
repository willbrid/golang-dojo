# Leçon 5 — Erreurs HTTP

> Dans un service, les erreurs naissent au fond — base de données, règle métier, validation — et
> doivent ressortir en haut sous la forme d'un **statut** et d'un **message** utiles au client. Deux
> fautes sont fréquentes : décider du statut à vingt endroits différents (incohérences garanties), et
> laisser filer le texte d'une erreur interne jusqu'au client (`pq: connection refused to
> 10.0.3.7:5432` : une adresse interne offerte à un attaquant). Cette leçon construit **un** modèle
> d'erreur, traduit **à un seul endroit**.

## Objectifs

1. Garder le domaine indépendant de HTTP : des erreurs **métier**, pas des statuts.
2. Traduire erreur → statut en un seul point, avec `errors.Is`/`errors.As`.
3. Rendre un corps d'erreur structuré (RFC 9457, *problem details*).
4. Ne jamais exposer un détail interne au client, mais toujours le journaliser.

## Explication

### Le domaine ne connaît pas HTTP

Le paquet métier (`account`) exprime **ce qui s'est passé** — compte introuvable, solde insuffisant,
entrée invalide — avec les outils du niveau 2 (leçon 6) : des **sentinelles** (`ErrNotFound`), des
**types d'erreur** (`*ValidationError`) qui portent des données, et l'enveloppement `%w` pour le
contexte. Il n'importe pas `net/http` et ne parle pas de 404. Le même domaine pourrait servir une CLI,
un service gRPC (leçon 12), une tâche de fond.

### La traduction, à un seul endroit

La couche HTTP traduit avec une fonction unique :

```go
switch {
case errors.As(err, &verr):                       return 422
case errors.Is(err, account.ErrNotFound):         return 404
case errors.Is(err, account.ErrInsufficient):     return 409
default:                                          return 500
}
```

`errors.Is` et `errors.As` traversent les enveloppes `%w` : le domaine peut ajouter tout le contexte
qu'il veut (`retrait de 1000 sur "alice" (solde 100) : solde insuffisant`) sans casser la traduction.

Le cas `default` est le plus important : **toute erreur non reconnue est une erreur interne**, 500,
sans détail. Une erreur de base de données, un fichier absent, un bug — le client n'a rien à en savoir,
et ne peut rien y faire.

### Des handlers qui **rendent** leur erreur

Avec `http.HandlerFunc`, chaque handler écrit lui-même ses erreurs (`http.Error` + `return`, leçon 2).
Un adaptateur maison inverse la responsabilité :

```go
type HandlerFunc func(w http.ResponseWriter, r *http.Request) error

func Adapt(h HandlerFunc) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if err := h(w, r); err != nil {
			writeProblem(w, err)        // la SEULE écriture d'erreur du service
		}
	})
}
```

Le handler fait `return err` comme n'importe quelle fonction Go ; plus de `return` oublié après un
`http.Error`, et le format d'erreur est uniforme par construction. C'est le patron le plus répandu
dans les services Go qui n'utilisent pas de framework.

Limite : si le handler a **déjà commencé** à écrire avant de rendre une erreur, il est trop tard pour
le statut (leçon 2). Règle : un handler qui rend une erreur n'a rien écrit.

### Le format : *problem details* (RFC 9457)

Plutôt que d'inventer un format, la RFC 9457 en propose un, avec le type de média
`application/problem+json` :

```json
{
  "type": "/problems/insufficient-funds",
  "title": "Solde insuffisant",
  "status": 409,
  "detail": "…",
  "fields": {"amount": "doit être positif"}
}
```

- `type` : un identifiant stable de la catégorie d'erreur (une URI, qui peut pointer vers une page de
  documentation) — c'est sur **lui** que les clients se branchent, jamais sur le texte ;
- `title` : un résumé lisible, identique pour toutes les occurrences du type ;
- `status` : le statut HTTP, répété dans le corps ;
- `detail` : une explication propre à **cette** occurrence (sans secret) ;
- des membres d'**extension** libres, comme `fields` pour les erreurs de validation champ par champ.

### Ne rien fuiter, tout journaliser

| Va au client | Va au journal |
|---|---|
| le type, le titre, le statut | le texte complet de l'erreur, avec son contexte |
| les erreurs de validation (c'est **son** entrée) | la pile (si panique) |
| un identifiant de requête (leçon 3) pour le support | l'identifiant de requête, pour relier les deux |

Jamais chez le client : texte d'erreur de base de données (requêtes SQL, noms de tables), chemins de
fichiers, adresses internes, versions de bibliothèques, piles d'appels. Ce sont des informations de
**reconnaissance** pour un attaquant.

Les erreurs 4xx sont des erreurs **du client** : inutile de les journaliser en erreur (au plus en
debug, ou en métrique). Les 5xx sont **nos** erreurs : journal + métrique + alerte.

### 404 ou 403 ?

Quand un utilisateur demande une ressource qui existe mais ne lui appartient pas, répondre **403**
révèle qu'elle existe. Pour des ressources sensibles, on répond **404** — comme si elle n'existait pas.
C'est une décision de sécurité, à prendre consciemment.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : account/account.go
// Package account est le DOMAINE : il ne connaît pas HTTP.
package account

import (
	"errors"
	"fmt"
)

var (
	ErrNotFound     = errors.New("compte introuvable")
	ErrInsufficient = errors.New("solde insuffisant")
)

// ValidationError décrit une entrée invalide, champ par champ.
type ValidationError struct {
	Fields map[string]string
}

func (e *ValidationError) Error() string { return fmt.Sprintf("entrée invalide : %v", e.Fields) }

type Service struct {
	balances map[string]int64
}

func NewService(b map[string]int64) *Service { return &Service{balances: b} }

func (s *Service) Withdraw(id string, amount int64) (int64, error) {
	fields := map[string]string{}
	if id == "" {
		fields["account"] = "requis"
	}
	if amount <= 0 {
		fields["amount"] = "doit être positif"
	}
	if len(fields) > 0 {
		return 0, &ValidationError{Fields: fields}
	}
	bal, ok := s.balances[id]
	if !ok {
		return 0, fmt.Errorf("retrait sur %q : %w", id, ErrNotFound)
	}
	if bal < amount {
		return 0, fmt.Errorf("retrait de %d sur %q (solde %d) : %w", amount, id, bal, ErrInsufficient)
	}
	if id == "panne" {
		return 0, errors.New("pq: connection refused to 10.0.3.7:5432") // une erreur d'infrastructure
	}
	s.balances[id] = bal - amount
	return s.balances[id], nil
}
```

```go
// Fichier : web/errors.go
// Package web traduit les erreurs du domaine en réponses HTTP.
package web

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"

	"exemple/account"
)

// Problem suit la forme « problem details » (RFC 9457).
type Problem struct {
	Type   string            `json:"type"`
	Title  string            `json:"title"`
	Status int               `json:"status"`
	Detail string            `json:"detail,omitempty"`
	Fields map[string]string `json:"fields,omitempty"`
}

// toProblem est le SEUL endroit où une erreur devient un statut.
func toProblem(err error) Problem {
	var verr *account.ValidationError
	switch {
	case errors.As(err, &verr):
		return Problem{Type: "/problems/validation", Title: "Entrée invalide", Status: http.StatusUnprocessableEntity, Fields: verr.Fields}
	case errors.Is(err, account.ErrNotFound):
		return Problem{Type: "/problems/not-found", Title: "Compte introuvable", Status: http.StatusNotFound}
	case errors.Is(err, account.ErrInsufficient):
		return Problem{Type: "/problems/insufficient-funds", Title: "Solde insuffisant", Status: http.StatusConflict}
	default:
		// Tout le reste est un problème interne : AUCUN détail ne part chez le client.
		return Problem{Type: "/problems/internal", Title: "Erreur interne", Status: http.StatusInternalServerError}
	}
}

// HandlerFunc est un handler qui RENVOIE son erreur au lieu de l'écrire.
type HandlerFunc func(w http.ResponseWriter, r *http.Request) error

// Adapt convertit un HandlerFunc en http.Handler : la réponse d'erreur est écrite ici, une fois.
func Adapt(log io.Writer, h HandlerFunc) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		err := h(w, r)
		if err == nil {
			return
		}
		p := toProblem(err)
		if p.Status >= 500 {
			fmt.Fprintf(log, "%s %s : %v\n", r.Method, r.URL.Path, err) // le détail va au JOURNAL
		}
		w.Header().Set("Content-Type", "application/problem+json")
		w.WriteHeader(p.Status)
		json.NewEncoder(w).Encode(p)
	})
}

type withdrawal struct {
	Account string `json:"account"`
	Amount  int64  `json:"amount"`
}

// Withdraw n'a qu'une préoccupation : traduire HTTP ↔ domaine. Les erreurs remontent.
func Withdraw(svc *account.Service) HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) error {
		var in withdrawal
		dec := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<12))
		dec.DisallowUnknownFields()
		if err := dec.Decode(&in); err != nil {
			return &account.ValidationError{Fields: map[string]string{"body": "JSON invalide"}}
		}
		bal, err := svc.Withdraw(in.Account, in.Amount)
		if err != nil {
			return err
		}
		w.Header().Set("Content-Type", "application/json")
		return json.NewEncoder(w).Encode(map[string]int64{"balance": bal})
	}
}
```

```go
// Fichier : web/errors_test.go
package web

import (
	"bytes"
	"net/http/httptest"
	"strings"
	"testing"

	"exemple/account"
)

func TestWithdraw(t *testing.T) {
	var log bytes.Buffer
	svc := account.NewService(map[string]int64{"alice": 100, "panne": 50})
	h := Adapt(&log, Withdraw(svc))
	tests := []struct {
		name, body string
		wantStatus int
		wantBody   string
	}{
		{"ok", `{"account":"alice","amount":30}`, 200, `{"balance":70}`},
		{"validation", `{"account":"","amount":-1}`, 422,
			`{"type":"/problems/validation","title":"Entrée invalide","status":422,"fields":{"account":"requis","amount":"doit être positif"}}`},
		{"introuvable", `{"account":"bob","amount":1}`, 404,
			`{"type":"/problems/not-found","title":"Compte introuvable","status":404}`},
		{"solde", `{"account":"alice","amount":1000}`, 409,
			`{"type":"/problems/insufficient-funds","title":"Solde insuffisant","status":409}`},
		{"interne", `{"account":"panne","amount":1}`, 500,
			`{"type":"/problems/internal","title":"Erreur interne","status":500}`},
		{"json", `{"amount":`, 422,
			`{"type":"/problems/validation","title":"Entrée invalide","status":422,"fields":{"body":"JSON invalide"}}`},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			rec := httptest.NewRecorder()
			h.ServeHTTP(rec, httptest.NewRequest("POST", "/withdraw", strings.NewReader(tt.body)))
			if got := strings.TrimSpace(rec.Body.String()); rec.Code != tt.wantStatus || got != tt.wantBody {
				t.Errorf("= %d %s\nattendu %d %s", rec.Code, got, tt.wantStatus, tt.wantBody)
			}
		})
	}
	if !strings.Contains(log.String(), "10.0.3.7") {
		t.Errorf("le détail interne doit être journalisé : %q", log.String())
	}
}
```

`go test -race ./...` passe. Remarquer la ligne « interne » : le client reçoit un 500 générique, et le
test vérifie que l'adresse `10.0.3.7` est dans le **journal**, et nulle part dans la réponse.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `account` n'importe pas `net/http` | Le domaine reste réutilisable et se teste sans HTTP. |
| `ValidationError` avec `Fields` | Toutes les erreurs d'entrée en une fois, champ par champ : le client corrige tout d'un coup. |
| `%w` dans le domaine | Le contexte (« retrait de 1000 sur alice ») est conservé pour le journal, sans gêner `errors.Is`. |
| `toProblem` | **Le** point de traduction. Ajouter un cas = une ligne, et tous les handlers en profitent. |
| `default` → 500 sans détail | L'erreur `pq: connection refused …` ne sort jamais vers le client. |
| Journal seulement pour les 5xx | Les erreurs client ne polluent pas le journal d'erreurs. |
| `Adapt` | Les handlers font `return err` ; l'écriture d'erreur est centralisée. |
| JSON illisible → `ValidationError` | Une seule forme d'erreur d'entrée pour le client. |
| `application/problem+json` | Le type de média de la RFC : les clients peuvent le reconnaître. |

## Erreurs fréquentes

1. **Des statuts HTTP dans le domaine** (`return 404`, ou `ErrNotFound` défini dans le paquet HTTP).
2. **`err.Error()` envoyé au client.**
3. **Comparer des messages d'erreur** (`strings.Contains(err.Error(), "not found")`) au lieu de `errors.Is`.
4. **Des formats d'erreur différents** selon les routes.
5. **Tout en 500**, ou tout en 400.
6. **Ne pas journaliser les 500**, ou les journaliser sans identifiant de requête.
7. **Écrire une partie de la réponse puis rendre une erreur.**
8. **403 qui révèle l'existence** d'une ressource privée.

## Bonnes pratiques Go

- Domaine : sentinelles et types d'erreur, `%w`, aucune notion de transport.
- Transport : une fonction unique erreur → statut + corps.
- Handlers qui rendent `error`, adaptés une fois.
- RFC 9457 pour le format ; `type` stable comme contrat.
- 5xx : journal complet + identifiant de requête ; client : message générique + identifiant.
- Un test par catégorie d'erreur, dont un qui vérifie l'**absence** de fuite.

## Ce que je dois retenir

- Le domaine dit **ce qui** s'est passé ; la couche HTTP décide du **statut**.
- Traduction **à un seul endroit**, par `errors.Is`/`errors.As` ; défaut = 500.
- Handlers `func(w, r) error` + adaptateur = réponses d'erreur uniformes.
- *Problem details* (RFC 9457) : `type`, `title`, `status`, `detail`, extensions.
- Aucun détail interne au client ; tout dans le journal.

---

➡️ [Exercices](exercices.md)
