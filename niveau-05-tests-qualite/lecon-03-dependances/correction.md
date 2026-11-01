# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Externe

Il suffit de changer la clause `package` en `reminder_test`, d'importer `exemple/reminder` et de
préfixer `New`, `Task`, `IsOverdue`. `fakeMailer` n'a **jamais** besoin de nommer le type
`mailer` : il a une méthode `Send(to, subject, body string) error`, il satisfait donc
l'interface, même déclarée non exportée dans un autre paquet. C'est la satisfaction implicite
(N2 L4) en action : le paquet `reminder` n'est couplé à **aucune** implémentation de courriel, ni
la vraie ni la fausse.

## E2 — Trouver les dépendances cachées

Cinq dépendances cachées : `os.Getenv` (environnement), `time.Now` (horloge), `os.Create`
(système de fichiers), `log.Printf` (sortie globale), `globalDB` (état global).

```go
type Backup struct {
	Dir   string                 // lu une fois dans main, depuis l'environnement
	Now   func() time.Time
	Open  func(name string) (io.WriteCloser, error)
	Log   io.Writer
	Data  archiveSource          // interface côté consommateur
}

func (b *Backup) Run() error {
	name := "backup-" + b.Now().Format("20060102") + ".tar"
	f, err := b.Open(filepath.Join(b.Dir, name))
	if err != nil {
		return err
	}
	fmt.Fprintf(b.Log, "sauvegarde dans %s\n", name)
	if err := writeArchive(f, b.Data); err != nil {
		f.Close()
		return err
	}
	return f.Close()   // erreur de Close vérifiée en écriture (N4 L7)
}
```

En test : `Now` figée, `Open` qui rend un `nopCloser` autour d'un `bytes.Buffer`, `Log` vers un
autre buffer. On pourrait aussi garder le vrai disque avec `Dir: t.TempDir()` et
`Open: func(n string) (io.WriteCloser, error) { return os.Create(n) }` — c'est souvent
**préférable** : le disque temporaire est rapide et réel. Le correctif caché : l'erreur de
`Close` était ignorée par le `defer` d'origine.

## E3 — Stub, fake, spy, mock

(a) **stub** ; (b) **fake** — une implémentation complète simplifiée ; (c) **spy** — il
enregistre les appels pour qu'on les examine après ; (d) **mock** — il porte lui-même les
attentes sur la séquence d'appels.

## E4 — L'erreur que le fake ne simule pas

Le service qui fait `u, _ := store.Get(id); return u.Name` passe tous les tests avec un fake qui
ne connaît pas `ErrNotFound`. Avec un fake correct (`map` + `ErrNotFound` pour une clé absente),
le test d'un identifiant inconnu révèle un nom vide rendu sans erreur. Le fake **définit ce que
le test peut voir** : un fake trop gentil rend les tests aveugles.

## E5 — Pas de `Sleep`

Deux raisons : le test dure au moins 150 ms (multiplié par des centaines de tests, la suite
devient lente) ; et il est **instable** sur une machine chargée (CI) où 50 ms de marge peuvent ne
pas suffire selon l'implémentation.

```go
type Cache struct {
	ttl  time.Duration
	now  func() time.Time
	data map[string]entry
}

func TestCacheExpires(t *testing.T) {
	now := time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC)
	c := NewCache(100*time.Millisecond, func() time.Time { return now })
	c.Set("k", "v")
	now = now.Add(150 * time.Millisecond)   // la closure voit la nouvelle valeur
	if _, ok := c.Get("k"); ok {
		t.Error("la clé aurait dû expirer")
	}
}
```

La closure **capture la variable** `now` (N1 L11) : avancer le temps, c'est modifier la
variable. Le test s'exécute en quelques microsecondes et ne dépend plus de la machine.

## Exercice intermédiaire — Le service de paiement

```go
// Package checkout encaisse une commande.
package checkout

import (
	"errors"
	"fmt"
	"io"
)

type Order struct {
	ID     string
	Amount int64 // centimes
	Email  string
}

type Card struct{ Token string }

type Receipt struct {
	OrderID  string
	ChargeID string
}

// Les trois besoins de Checkout, une interface chacun.
type gateway interface {
	Charge(idempotencyKey string, card Card, amount int64) (chargeID string, err error)
	Refund(chargeID string) error
}

type orderStore interface {
	SavePaid(o Order, chargeID string) error
}

type receiptSender interface {
	SendReceipt(email string, r Receipt) error
}

type Checkout struct {
	gw     gateway
	orders orderStore
	mail   receiptSender
	warn   io.Writer // avertissements non bloquants (journal)
}

func New(gw gateway, orders orderStore, mail receiptSender, warn io.Writer) *Checkout {
	return &Checkout{gw: gw, orders: orders, mail: mail, warn: warn}
}

var ErrDeclined = errors.New("paiement refusé")

func (c *Checkout) Pay(o Order, card Card) (Receipt, error) {
	chargeID, err := c.gw.Charge(o.ID, card, o.Amount) // l'ID de commande sert de clé d'idempotence
	if err != nil {
		return Receipt{}, fmt.Errorf("commande %s : débit : %w", o.ID, err)
	}
	if err := c.orders.SavePaid(o, chargeID); err != nil {
		if rerr := c.gw.Refund(chargeID); rerr != nil {
			// le pire cas : débité, non enregistré, non remboursé — à remonter avec les deux causes
			return Receipt{}, fmt.Errorf("commande %s : enregistrement : %w ; remboursement : %w", o.ID, err, rerr)
		}
		return Receipt{}, fmt.Errorf("commande %s : enregistrement (remboursée) : %w", o.ID, err)
	}
	r := Receipt{OrderID: o.ID, ChargeID: chargeID}
	if err := c.mail.SendReceipt(o.Email, r); err != nil {
		fmt.Fprintf(c.warn, "commande %s : reçu non envoyé : %v\n", o.ID, err)
	}
	return r, nil
}
```

```go
package checkout

import (
	"errors"
	"fmt"
	"io"
	"testing"
)

type fakeGateway struct {
	decline  bool
	charges  map[string]string // clé d'idempotence → chargeID
	debited  map[string]int64  // chargeID → montant (débits effectifs)
	refunded map[string]bool
}

func newFakeGateway() *fakeGateway {
	return &fakeGateway{charges: map[string]string{}, debited: map[string]int64{}, refunded: map[string]bool{}}
}

func (g *fakeGateway) Charge(key string, _ Card, amount int64) (string, error) {
	if g.decline {
		return "", ErrDeclined
	}
	if id, ok := g.charges[key]; ok {
		return id, nil // idempotent : même clé, même débit, pas de second prélèvement
	}
	id := fmt.Sprintf("ch_%d", len(g.charges)+1)
	g.charges[key] = id
	g.debited[id] = amount
	return id, nil
}

func (g *fakeGateway) Refund(id string) error { g.refunded[id] = true; return nil }

type fakeOrders struct {
	fail error
	paid map[string]string
}

func (s *fakeOrders) SavePaid(o Order, ch string) error {
	if s.fail != nil {
		return s.fail
	}
	if s.paid == nil {
		s.paid = map[string]string{}
	}
	s.paid[o.ID] = ch
	return nil
}

type fakeMail struct {
	fail error
	sent []Receipt
}

func (m *fakeMail) SendReceipt(_ string, r Receipt) error {
	if m.fail != nil {
		return m.fail
	}
	m.sent = append(m.sent, r)
	return nil
}

var errDB = errors.New("base indisponible")

func TestPay(t *testing.T) {
	order := Order{ID: "o1", Amount: 4990, Email: "awa@x.fr"}
	tests := []struct {
		name         string
		decline      bool
		dbErr        error
		mailErr      error
		payTwice     bool
		wantErr      error
		wantDebited  int // nombre de débits effectifs
		wantRefunded bool
		wantSaved    bool
		wantMails    int
	}{
		{name: "succès", wantDebited: 1, wantSaved: true, wantMails: 1},
		{name: "refusé", decline: true, wantErr: ErrDeclined},
		{name: "base KO → remboursé", dbErr: errDB, wantErr: errDB, wantDebited: 1, wantRefunded: true},
		{name: "reçu KO → payé quand même", mailErr: errors.New("smtp"), wantDebited: 1, wantSaved: true},
		{name: "double paiement", payTwice: true, wantDebited: 1, wantSaved: true, wantMails: 2},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			gw := newFakeGateway()
			gw.decline = tt.decline
			st := &fakeOrders{fail: tt.dbErr}
			ml := &fakeMail{fail: tt.mailErr}
			c := New(gw, st, ml, io.Discard)

			_, err := c.Pay(order, Card{Token: "tok"})
			if tt.payTwice {
				_, err = c.Pay(order, Card{Token: "tok"})
			}
			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("Pay : erreur = %v, attendu %v", err, tt.wantErr)
			}
			if len(gw.debited) != tt.wantDebited {
				t.Errorf("débits = %d, attendu %d", len(gw.debited), tt.wantDebited)
			}
			if got := len(gw.refunded) > 0; got != tt.wantRefunded {
				t.Errorf("remboursé = %v, attendu %v", got, tt.wantRefunded)
			}
			if _, got := st.paid[order.ID]; got != tt.wantSaved {
				t.Errorf("commande enregistrée = %v, attendu %v", got, tt.wantSaved)
			}
			if len(ml.sent) != tt.wantMails {
				t.Errorf("reçus envoyés = %d, attendu %d", len(ml.sent), tt.wantMails)
			}
		})
	}
}
```

Points de correction :

- **Trois interfaces, trois méthodes au plus.** `gateway` a `Charge` **et** `Refund` parce que
  `Checkout` utilise les deux ; il n'a pas `Capture`, `ListCharges`… même si la vraie passerelle
  les offre. Chaque méthode d'interface doit être **utilisée** par le consommateur.
- **Le cas le plus grave** — débité, enregistrement raté, remboursement raté — est remonté avec
  **les deux** causes (`%w` deux fois, Go 1.20). Ce cas-là exige une intervention humaine ; le
  niveau 10 (outbox, réconciliation) apprendra à le rendre récupérable automatiquement.
- **Le reçu raté n'est pas une erreur** pour l'appelant : il est journalisé. Le test le vérifie
  en exigeant `err == nil`, commande enregistrée, zéro reçu.
- **Les tests vérifient l'état final** des fakes (combien de débits, remboursé ou non, commande
  enregistrée, reçus). Si `Pay` est réécrit pour enregistrer **avant** de débiter (stratégie
  différente, même contrat), seuls les cas dont le résultat change réellement casseront.
- **Le fake de passerelle doit implémenter l'idempotence** : c'est une propriété du **contrat**
  de la passerelle (on lui passe une clé), que `Checkout` exploite. Sans elle dans le fake, le cas
  « double paiement » montrerait **deux** débits et l'on croirait que `Checkout` est fautif ; ou
  bien on testerait que `Checkout` appelle `Charge` une seule fois — ce qui serait un test de
  mock, sur l'implémentation. Le fake reproduit le comportement réel ; le test vérifie qu'en fin
  de compte la carte n'est débitée qu'une fois.
- `log/slog` (N12 L6) serait le choix idiomatique pour l'avertissement ; un `io.Writer` suffit ici
  et se teste avec un `bytes.Buffer`.

## Défi — Le mock qui casse au refactoring

1. Le test avec `mockgen` : `m.EXPECT().Send("awa@x.fr", gomock.Any(), gomock.Any()).Return(nil).Times(1)`, etc.
2. Après le passage à un courriel récapitulatif par destinataire : le test **à mock casse** (il
   attendait un appel par tâche), le test **à fake** casse aussi s'il vérifiait le nombre de
   messages, mais pas s'il vérifiait « chaque destinataire en retard a été prévenu ». Le
   comportement observable voulu (« chaque propriétaire est prévenu de ses retards ») n'a pas
   changé : **aucun** des deux n'aurait dû casser. Le test qui vérifie ce comportement-là est
   celui du fake bien écrit.
3. Le mock détecte facilement un appel **interdit** : « `Send` ne doit **jamais** être appelé pour
   une tâche non échue » (un `.Times(0)`). Le fake peut le détecter aussi : vérifier que la liste
   des destinataires ne contient **pas** `ines@x.fr`. Tout ce qu'un mock vérifie, un spy peut le
   vérifier — mais **après coup**, et seulement ce qu'on choisit de regarder.
4. Un mock est le bon outil quand **l'interaction elle-même est le contrat** : un protocole
   (« `Begin` puis `Commit` ou `Rollback`, jamais les deux »), un appel qui ne doit pas avoir lieu
   (pas d'envoi en mode simulation), une interface tierce énorme qu'on ne peut pas simuler
   fidèlement. Dans tous les autres cas, vérifier un résultat avec un fake donne des tests plus
   robustes.

## Quiz

1. Une dépendance que la fonction va chercher elle-même au lieu de la recevoir : `time.Now`,
   `os.Getenv`, une variable globale, `http.DefaultClient`, `os.Stdout`.
2. Un paramètre.
3. Parce qu'elle décrit exactement le besoin, reste petite, et ne crée aucune dépendance du
   fournisseur vers le consommateur.
4. Un fake est une implémentation simplifiée fonctionnelle ; un mock vérifie lui-même la
   séquence d'appels reçus.
5. Parce qu'il vérifie **comment** le code fait (appels, ordre, nombre) au lieu de **ce qu'il
   produit**.
6. Les erreurs et cas limites de la dépendance réelle (introuvable, panne, conflit).
7. Ce qui est rapide, déterministe et isolable : fichiers temporaires, buffers, ses propres
   fonctions pures.
8. Il ralentit la suite et la rend instable ; il faut injecter l'horloge.
9. L'état global est partagé : tests non parallélisables, effets de bord d'un test sur l'autre,
   oubli de restauration.
10. `testing/synctest`.
