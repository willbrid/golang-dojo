# Leçon 7 — Monolithe modulaire et microservices

> « Faut-il des microservices ? » est souvent posé trop tôt, et pour de mauvaises raisons. Un appel de
> fonction coûte quelques centaines de nanosecondes et ne tombe jamais en panne ; le même appel à travers
> le réseau coûte des dizaines de microsecondes, peut échouer de mille façons, et oblige à versionner une
> API. Cette leçon défend un défaut raisonnable — le **monolithe modulaire** : un seul binaire, découpé en
> modules aux frontières **vérifiées** — et donne les critères qui justifient d'en extraire un service.

## Objectifs

1. Découper un monolithe en modules : une API publique par module, des détails dans `internal/`.
2. Laisser le compilateur et les tests **vérifier** les frontières.
3. Isoler les données de chaque module.
4. Mesurer ce que coûte une frontière réseau.
5. Savoir quand extraire un module en service, et comment préparer l'extraction.

## Explication

### Le monolithe modulaire

Un seul programme, un seul déploiement, mais découpé en **modules** qui correspondent aux contextes bornés
de la leçon 6 (catalogue, prêt, adhérents, facturation…). Chaque module :

- expose une **API publique** (un paquet, ici `catalog`) : les seuls types et fonctions que les autres
  modules ont le droit d'utiliser ;
- cache tout le reste dans `catalog/internal/…` ;
- **possède ses données** : ses tables (idéalement un schéma PostgreSQL par module), que les autres ne lisent
  ni n'écrivent jamais directement ;
- communique avec les autres par leur API publique (appels de fonction) ou par **événements** (leçon 8).

On obtient l'essentiel des bénéfices d'une architecture de services (équipes et code indépendants,
frontières claires, possibilité d'extraction future) **sans** leurs coûts (réseau, déploiements
coordonnés, transactions distribuées).

### `internal/` : une frontière que le compilateur garde

En Go, un paquet dont le chemin contient `internal` ne peut être importé que par le code situé **sous le
parent** de ce `internal` (N2 L8). `exemple/catalog/internal/store` est importable par `exemple/catalog`
et ses sous-paquets, **pas** par `exemple/lending`. Mesuré :

```
package exemple/lending
	lending/hack.go:3:8: use of internal package exemple/catalog/internal/store not allowed
```

C'est un outil d'architecture de premier ordre, propre à Go : la frontière n'est pas une convention que
chacun doit respecter, c'est une erreur de compilation.

Ce qu'`internal/` ne garde **pas** : les imports entre APIs publiques (le prêt peut importer `catalog`,
et le catalogue `lending`, sans cycle direct) ni l'accès aux tables d'un autre module. Pour ces règles-là,
un **test d'architecture** (`go list -deps`, comme au défi de la leçon 6) et, côté base, des **droits**
(un rôle PostgreSQL par module, qui n'a accès qu'à son schéma).

### Les données appartiennent au module

La règle la plus importante, et la plus souvent violée : **un module ne lit pas les tables d'un autre**.
Une jointure SQL entre `lending.loans` et `catalog.works` couple les deux modules plus sûrement que
n'importe quel import : on ne peut plus changer le schéma du catalogue sans casser le prêt, ni extraire
le catalogue sans réécrire la requête. Les alternatives : appeler l'API du module, ou garder dans le
module consommateur une **copie** des données dont il a besoin, tenue à jour par événements (leçon 8).

### Ce que coûte une frontière réseau

L'exemple consomme le **même** module catalogue de deux façons : par appel de fonction, et par HTTP sur la
machine locale (le meilleur cas réseau possible). Mesuré :

| Assemblage | Temps par reçu | Allocations |
|---|---|---|
| en mémoire | ≈ 190 ns | 3 |
| HTTP local | ≈ 80 µs (≈ 420 fois plus) | 83 |

Et ce n'est que la partie mesurable. Une frontière réseau ajoute aussi :

- des **pannes partielles** : le service est tombé, lent, ou répond une fois sur deux (`connection
  refused` dans l'exemple — une erreur qui n'existait pas en mémoire) ; il faut délais, réessais,
  disjoncteurs (leçon 9) ;
- la perte des **transactions** : deux modules en mémoire partagent une transaction ; deux services
  non. La cohérence devient « à terme », par événements et compensations (leçon 8) ;
- le **versionnement** d'API : on ne déploie plus les deux côtés ensemble (N7 L4, niveau 13) ;
- l'**observabilité** distribuée : une requête traverse plusieurs processus (leçon 10) ;
- l'**exploitation** : un déploiement, une configuration, une surveillance de plus par service.

### Quand extraire un service

Des raisons qui tiennent :

- un module a des **besoins de ressources** très différents (calcul intensif, mémoire énorme) et doit
  être mis à l'échelle seul ;
- une **équipe** distincte possède le module, avec son propre rythme de déploiement, et le couplage de
  déploiement coûte réellement ;
- un **besoin d'isolation** : sécurité (paiements, données de santé), tolérance aux pannes (un module
  fragile ne doit pas faire tomber le reste) ;
- une **technologie** différente est réellement nécessaire.

Des raisons qui ne tiennent pas : « c'est moderne », « pour que le code soit propre » (les modules le
font), « pour scaler » sans mesure, « une équipe par service » quand il n'y a qu'une équipe.

Le monolithe modulaire **prépare** l'extraction : si les modules communiquent par des interfaces
déclarées côté consommateur (leçon 4) et ne partagent pas de tables, extraire un module revient à écrire
un adaptateur HTTP ou gRPC côté fournisseur et un client côté consommateur — et à **changer
l'assemblage** dans `main`. L'exemple le montre : le module de prêt ne change pas d'une ligne.

### Quand ne pas le faire

- **Un module par entité** (`users`, `books`, `loans` séparés alors qu'ils forment un seul contexte) : des
  appels incessants entre modules pour la moindre opération.
- **Des microservices dès le départ** pour un produit dont on ne connaît pas encore les frontières : elles
  bougeront, et déplacer une frontière entre services coûte bien plus qu'entre paquets.
- **Un monolithe « modulaire » qui partage ses tables** : il a le couplage du monolithe et la complexité
  des modules.

## Exemple

Deux modules : le catalogue (API publique + détail `internal`) et le prêt, qui déclare son besoin. Les
tests montrent le même comportement en mémoire et à travers HTTP, une panne qui n'existe qu'à travers le
réseau, et le benchmark mesure l'écart.

```go
// Fichier : catalog/internal/store/store.go
// Détail interne du module catalogue. Le compilateur interdit tout import depuis
// l'extérieur de catalog/ : la règle « internal » de Go.
package store

import "sync"

type Store struct {
	mu     sync.RWMutex
	titles map[string]string
}

func New(seed map[string]string) *Store { return &Store{titles: seed} }

func (s *Store) Title(isbn string) (string, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	t, ok := s.titles[isbn]
	return t, ok
}
```

```go
// Fichier : catalog/catalog.go
// Package catalog : l'API PUBLIQUE du module. C'est tout ce que les autres modules voient.
package catalog

import (
	"context"
	"errors"

	"exemple/catalog/internal/store"
)

var ErrUnknownWork = errors.New("œuvre inconnue")

type Module struct{ store *store.Store }

func New(seed map[string]string) *Module { return &Module{store: store.New(seed)} }

func (m *Module) Title(_ context.Context, isbn string) (string, error) {
	if t, ok := m.store.Title(isbn); ok {
		return t, nil
	}
	return "", ErrUnknownWork
}
```

```go
// Fichier : catalog/httpapi/httpapi.go
// Le même module, exposé en HTTP : ce qu'on écrirait le jour où on l'extrait en service.
package httpapi

import (
	"encoding/json"
	"errors"
	"net/http"

	"exemple/catalog"
)

func Handler(m *catalog.Module) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /works/{isbn}", func(w http.ResponseWriter, r *http.Request) {
		t, err := m.Title(r.Context(), r.PathValue("isbn"))
		if errors.Is(err, catalog.ErrUnknownWork) {
			http.Error(w, err.Error(), http.StatusNotFound)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"title": t})
	})
	return mux
}
```

```go
// Fichier : lending/lending.go
// Package lending : un autre module. Il déclare ce dont il a besoin du catalogue,
// sans savoir si la réponse viendra d'un appel de fonction ou du réseau.
package lending

import (
	"context"
	"fmt"
)

type titleSource interface {
	Title(ctx context.Context, isbn string) (string, error)
}

type Module struct{ titles titleSource }

func New(t titleSource) *Module { return &Module{titles: t} }

// Receipt : le reçu d'emprunt, qui affiche le titre de l'œuvre.
func (m *Module) Receipt(ctx context.Context, member, isbn string) (string, error) {
	t, err := m.titles.Title(ctx, isbn)
	if err != nil {
		return "", fmt.Errorf("reçu pour %s : %w", isbn, err)
	}
	return fmt.Sprintf("%s a emprunté « %s »", member, t), nil
}
```

```go
// Fichier : lending/catalogclient/client.go
// Client HTTP du catalogue : satisfait l'interface titleSource de lending.
package catalogclient

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
)

type Client struct {
	Base string
	HTTP *http.Client
}

func (c Client) Title(ctx context.Context, isbn string) (string, error) {
	req, err := http.NewRequestWithContext(ctx, "GET", c.Base+"/works/"+url.PathEscape(isbn), nil)
	if err != nil {
		return "", err
	}
	resp, err := c.HTTP.Do(req)
	if err != nil {
		return "", err // une erreur qui n'existait pas en mémoire : le réseau
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return "", fmt.Errorf("catalogue : statut %d", resp.StatusCode)
	}
	var body struct {
		Title string `json:"title"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
		return "", err
	}
	return body.Title, nil
}
```

```go
// Fichier : lending/lending_test.go
package lending_test

import (
	"context"
	"net/http"
	"net/http/httptest"
	"testing"

	"exemple/catalog"
	"exemple/catalog/httpapi"
	"exemple/lending"
	"exemple/lending/catalogclient"
)

var seed = map[string]string{"9782070360420": "La Peste"}

// Les deux assemblages : le même module catalogue, en mémoire ou derrière HTTP.
func inProcess(testing.TB) *lending.Module { return lending.New(catalog.New(seed)) }

func overHTTP(tb testing.TB) *lending.Module {
	srv := httptest.NewServer(httpapi.Handler(catalog.New(seed)))
	tb.Cleanup(srv.Close)
	return lending.New(catalogclient.Client{Base: srv.URL, HTTP: srv.Client()})
}

func TestSameBehaviour(t *testing.T) {
	for name, build := range map[string]func(testing.TB) *lending.Module{"mémoire": inProcess, "HTTP": overHTTP} {
		t.Run(name, func(t *testing.T) {
			got, err := build(t).Receipt(t.Context(), "ada", "9782070360420")
			if err != nil || got != "ada a emprunté « La Peste »" {
				t.Fatalf("%q %v", got, err)
			}
			if _, err := build(t).Receipt(t.Context(), "ada", "000"); err == nil {
				t.Fatal("œuvre inconnue : erreur attendue")
			}
		})
	}
}

func TestOnlyHTTPCanFailThisWay(t *testing.T) {
	srv := httptest.NewServer(http.NotFoundHandler())
	srv.Close() // le service est tombé
	m := lending.New(catalogclient.Client{Base: srv.URL, HTTP: srv.Client()})
	_, err := m.Receipt(context.Background(), "ada", "9782070360420")
	t.Log(err)
	if err == nil {
		t.Fatal("erreur réseau attendue")
	}
}

func BenchmarkReceipt(b *testing.B) {
	for name, build := range map[string]func(testing.TB) *lending.Module{"memoire": inProcess, "http": overHTTP} {
		b.Run(name, func(b *testing.B) {
			m := build(b)
			for b.Loop() {
				if _, err := m.Receipt(b.Context(), "ada", "9782070360420"); err != nil {
					b.Fatal(err)
				}
			}
		})
	}
}
```
`go test -race ./...` passe. La panne simulée produit :

```
reçu pour 9782070360420 : Get "http://127.0.0.1:43571/works/9782070360420": dial tcp 127.0.0.1:43571: connect: connection refused
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `catalog/internal/store` | Le détail du module ; importable seulement depuis `catalog/…`. |
| `catalog.Module` | L'API publique : ce que les autres modules ont le droit d'utiliser. |
| `catalog/httpapi` | Le même module exposé en HTTP : l'adaptateur qu'on écrit le jour de l'extraction. |
| `lending.titleSource` | Le besoin du prêt, déclaré côté consommateur : appel de fonction ou client HTTP, peu importe. |
| `catalogclient.Client` | Satisfait `titleSource` en passant par le réseau ; ajoute ses propres erreurs. |
| `inProcess` / `overHTTP` | Deux assemblages ; le module de prêt est identique. |
| `TestSameBehaviour` | La même suite contre les deux : le comportement fonctionnel est conservé. |
| `TestOnlyHTTPCanFailThisWay` | Une classe d'erreurs nouvelle, que le code appelant doit désormais gérer. |
| `BenchmarkReceipt` | ≈ 190 ns contre ≈ 80 µs : le coût minimal d'une frontière réseau. |

## Erreurs fréquentes

1. **Des microservices pour des raisons de mode** ou d'« organisation du code ».
2. **Des modules qui lisent les tables des autres** (jointures entre modules).
3. **Pas de frontière vérifiée** : la modularité tient à la discipline de chacun, et s'érode.
4. **Un découpage par entité technique** plutôt que par contexte métier.
5. **Extraire un service sans délais ni réessais** côté client (leçon 9).
6. **Croire qu'un appel réseau se comporte comme un appel de fonction** : latence, pannes partielles,
   absence de transaction.
7. **Un « monolithe distribué »** : des services qui doivent être déployés ensemble, qui partagent une base,
   ou s'appellent en chaîne synchrone pour chaque requête.

## Bonnes pratiques Go

- Un module par contexte borné ; une API publique ; le reste dans `internal/`.
- Frontières vérifiées : `internal/` par le compilateur, un test d'architecture pour le reste.
- Un schéma de base par module, aucun accès croisé.
- Communication par interfaces côté consommateur et par événements.
- Extraire un service sur des critères mesurables (ressources, équipes, isolation), et préparer l'extraction
  par la conception plutôt que de la faire tôt.

## Ce que je dois retenir

- Le monolithe modulaire est un défaut raisonnable : frontières de services, coûts d'un seul programme.
- `internal/` est une frontière gardée par le compilateur.
- Les données appartiennent à leur module ; pas de jointure entre modules.
- Une frontière réseau coûte des centaines de fois plus cher, et apporte pannes, versions et cohérence à
  terme.
- Bien découpé, un module s'extrait en changeant l'assemblage, pas le code des autres modules.

---

➡️ [Exercices](exercices.md)
