# Leçon 3 — Tester les dépendances

> Une fonction qui calcule est facile à tester. Une fonction qui lit l'heure, écrit sur le
> disque, envoie un courriel ou appelle une API l'est beaucoup moins : le résultat change
> d'une exécution à l'autre, le test est lent, ou il a des effets réels. Cette leçon montre
> comment **isoler** ces dépendances — et pourquoi la communauté Go préfère les **fakes**
> écrits à la main aux frameworks de mocks.

## Objectifs

1. Reconnaître une dépendance cachée qui rend un code intestable.
2. L'injecter : paramètre, champ, interface côté consommateur, fonction.
3. Écrire des **fakes** et savoir ce qui les distingue des **stubs** et des **mocks**.
4. Choisir ce qui mérite un double de test, et ce qui doit rester réel.

## Explication

### Les dépendances cachées

```go
func IsOverdue(t Task) bool {
	return time.Now().After(t.Due)          // ← dépend de l'instant où le test tourne
}

func SendReminders(tasks []Task) error {
	for _, t := range tasks {
		if IsOverdue(t) {
			smtp.SendMail("smtp.example.com:25", …)   // ← envoie un VRAI courriel
		}
	}
	return nil
}
```

Pour tester `IsOverdue`, il faudrait une échéance relative à « maintenant »… qui change entre la
construction du cas et la vérification. Pour tester `SendReminders`, il faudrait un serveur de
courriel. Les deux fonctions **cachent** leurs dépendances.

La règle de fond : **une fonction devrait recevoir tout ce dont elle a besoin**. Ce qu'elle va
chercher elle-même (`time.Now`, `os.Getenv`, une variable globale, `http.DefaultClient`) est
une dépendance cachée.

### Quatre façons d'injecter

| Technique | Exemple | Quand |
|---|---|---|
| **Paramètre** | `IsOverdue(t Task, now time.Time) bool` | la plus simple ; toujours la première à envisager |
| **Champ de struct** | `type Service struct { now func() time.Time }` | la dépendance sert dans plusieurs méthodes |
| **Interface côté consommateur** | `type mailer interface { Send(to, body string) error }` | un comportement avec plusieurs opérations, ou un état |
| **Valeur de fonction** | `type sendFunc func(to, body string) error` | une seule opération : plus léger qu'une interface |

`IsOverdue(t, now)` est l'exemple parfait : la fonction devient **pure** — mêmes entrées,
même résultat — et son test tient en une ligne de table. C'est la contrainte 4 du projet 2.

L'interface **côté consommateur** (N2 L4) prend ici tout son sens : le paquet qui envoie les
rappels déclare une interface `mailer` d'**une** méthode, exactement ce dont il a besoin ; le
vrai client SMTP la satisfait sans le savoir, et le test fournit un faux de trois lignes.

### Le vocabulaire des doubles de test

| Double | Ce qu'il fait | Exemple |
|---|---|---|
| **Stub** | rend des réponses préprogrammées | une horloge qui rend toujours 10 h |
| **Fake** | une **vraie implémentation simplifiée** | un stockage en `map` à la place de PostgreSQL ; un `fstest.MapFS` |
| **Spy** | un stub qui **enregistre** les appels pour vérification | un `mailer` qui garde la liste des messages « envoyés » |
| **Mock** | un objet qui **vérifie lui-même** qu'il a été appelé comme prévu | généré par un outil, avec des attentes du type « Send doit être appelé une fois avec ces arguments » |

### Pourquoi Go préfère les fakes

Dans beaucoup d'écosystèmes, le réflexe est un framework de mocks. En Go, ils existent
(`go.uber.org/mock`, `testify/mock`), mais la culture dominante les évite, pour trois raisons :

1. **Les interfaces sont petites** : un fake d'une interface d'une ou deux méthodes s'écrit en
   cinq lignes. Le framework coûte plus qu'il ne rapporte.
2. **Un mock teste l'implémentation** : « `Send` a été appelé exactement une fois, avec ces
   arguments, avant `Close` ». Refactoriser sans changer le comportement (grouper deux envois)
   casse le test. Un fake teste le **résultat** : « après l'appel, la boîte contient un message
   pour Awa ».
3. **Un fake est réutilisable** dans tous les tests, et peut même servir en développement local
   (un stockage en mémoire pour lancer le service sans base de données).

Un bon fake est **correct** pour ce qu'il simule : un fake de stockage qui ne renvoie jamais
`ErrNotFound` cache les bugs de gestion d'erreur. Si le fake devient compliqué, c'est souvent
que l'interface est trop grosse.

### Ce qu'il ne faut pas doubler

Doubler, c'est tester **contre une hypothèse** sur le comportement de la dépendance. Si
l'hypothèse est fausse, le test passe et la production casse. Donc :

- **Ne pas doubler ce qu'on peut utiliser réellement à faible coût** : le système de fichiers
  avec `t.TempDir()` (rapide, isolé), un `bytes.Buffer` comme `io.Writer`, `strings.NewReader`.
- **Ne pas doubler ses propres types** de calcul pur : les appeler.
- **Doubler** : le temps, l'aléatoire, le réseau, les services externes (courriel, paiement),
  ce qui est lent ou non déterministe.
- **Compléter** par quelques tests d'intégration contre la vraie dépendance (base de données
  réelle au niveau 8 avec `testcontainers`), lancés moins souvent.

### Le temps, cas particulier important

Trois façons d'injecter l'horloge, de la plus simple à la plus complète :

```go
// 1. Paramètre : pour les fonctions pures
func IsOverdue(t Task, now time.Time) bool

// 2. Champ fonction : pour un service
type Service struct{ now func() time.Time }        // en production : time.Now

// 3. Interface : quand il faut aussi des minuteries (After, Ticker)
type Clock interface {
	Now() time.Time
	After(d time.Duration) <-chan time.Time          // les channels arrivent au niveau 6
}
```

Ne **jamais** écrire `time.Sleep` dans un test pour « attendre que le temps passe » : le test
devient lent et instable. Pour le code concurrent qui utilise de vraies minuteries, Go 1.25 a
ajouté `testing/synctest`, qui fait avancer une horloge **virtuelle** — on le verra au niveau 6.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : reminder/reminder.go
// Package reminder envoie des rappels pour les tâches en retard.
package reminder

import (
	"errors"
	"fmt"
	"time"
)

type Task struct {
	ID    int
	Title string
	Owner string
	Due   time.Time
}

// mailer : ce dont ce paquet a besoin, et rien de plus. Non exporté.
type mailer interface {
	Send(to, subject, body string) error
}

// Service envoie les rappels. Ses dépendances sont des champs, fournis par New.
type Service struct {
	mail mailer
	now  func() time.Time
}

func New(m mailer, now func() time.Time) *Service {
	return &Service{mail: m, now: now}
}

// IsOverdue est pure : l'instant de référence est un paramètre.
func IsOverdue(t Task, now time.Time) bool {
	return !t.Due.IsZero() && now.After(t.Due)
}

// Remind envoie un rappel par tâche en retard ; un échec n'arrête pas les autres envois.
func (s *Service) Remind(tasks []Task) (sent int, err error) {
	now := s.now()
	var errs []error
	for _, t := range tasks {
		if !IsOverdue(t, now) {
			continue
		}
		late := now.Sub(t.Due).Round(time.Hour)
		body := fmt.Sprintf("La tâche %q est en retard de %v.", t.Title, late)
		if err := s.mail.Send(t.Owner, "Rappel : "+t.Title, body); err != nil {
			errs = append(errs, fmt.Errorf("tâche %d : %w", t.ID, err))
			continue
		}
		sent++
	}
	return sent, errors.Join(errs...)
}
```

```go
// Fichier : reminder/reminder_test.go
package reminder

import (
	"errors"
	"strings"
	"testing"
	"time"
)

// fakeMailer est un spy : il enregistre les messages au lieu de les envoyer,
// et peut simuler une panne pour certains destinataires.
type fakeMailer struct {
	sent   []string          // destinataires, dans l'ordre
	bodies map[string]string // dernier corps par destinataire
	failTo map[string]bool
}

var errSMTP = errors.New("serveur indisponible")

func (f *fakeMailer) Send(to, subject, body string) error {
	if f.failTo[to] {
		return errSMTP
	}
	f.sent = append(f.sent, to)
	if f.bodies == nil {
		f.bodies = map[string]string{}
	}
	f.bodies[to] = body
	return nil
}

var noon = time.Date(2026, 11, 2, 12, 0, 0, 0, time.UTC)

func TestIsOverdue(t *testing.T) {
	tests := []struct {
		name string
		due  time.Time
		want bool
	}{
		{"hier", noon.Add(-24 * time.Hour), true},
		{"une seconde avant", noon.Add(-time.Second), true},
		{"pile maintenant", noon, false},
		{"demain", noon.Add(24 * time.Hour), false},
		{"sans échéance", time.Time{}, false},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := IsOverdue(Task{Due: tt.due}, noon); got != tt.want {
				t.Errorf("IsOverdue(due=%v, now=%v) = %v, attendu %v", tt.due, noon, got, tt.want)
			}
		})
	}
}

func TestRemind(t *testing.T) {
	m := &fakeMailer{failTo: map[string]bool{"bob@x.fr": true}}
	s := New(m, func() time.Time { return noon }) // l'horloge est figée

	tasks := []Task{
		{ID: 1, Title: "Rapport", Owner: "awa@x.fr", Due: noon.Add(-50 * time.Hour)},
		{ID: 2, Title: "Facture", Owner: "bob@x.fr", Due: noon.Add(-time.Hour)},
		{ID: 3, Title: "Plus tard", Owner: "ines@x.fr", Due: noon.Add(time.Hour)},
	}
	sent, err := s.Remind(tasks)

	if sent != 1 {
		t.Errorf("sent = %d, attendu 1", sent)
	}
	if !errors.Is(err, errSMTP) {
		t.Errorf("err = %v, attendu une erreur enveloppant errSMTP", err)
	}
	if len(m.sent) != 1 || m.sent[0] != "awa@x.fr" {
		t.Errorf("destinataires = %v, attendu [awa@x.fr]", m.sent)
	}
	if body := m.bodies["awa@x.fr"]; !strings.Contains(body, "en retard de 50h0m0s") {
		t.Errorf("corps = %q, attendu la mention du retard", body)
	}
}
```

Sortie de `go test ./...` :

```
ok  	exemple/reminder	0.003s
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `mailer` non exportée, une méthode | Définie par le consommateur ; le vrai client SMTP la satisfait sans en dépendre. |
| `now func() time.Time` | En production : `New(smtpClient, time.Now)`. En test : une closure qui rend un instant fixe. |
| `IsOverdue(t, now)` | Pure : testée par une table, sans aucun double. |
| `now := s.now()` **une fois** | Toutes les tâches sont évaluées au même instant : pas d'incohérence si la boucle dure. |
| `fakeMailer` | Un spy de 15 lignes, sans framework. Il simule aussi la **panne**, pour tester le chemin d'erreur. |
| `errors.Join` + `%w` | `errors.Is(err, errSMTP)` traverse le `Join` **et** l'enveloppe. |
| Ce qui est vérifié | Le **résultat** (qui a reçu quoi), pas la séquence d'appels : regrouper les envois plus tard ne cassera pas ce test. |
| `package reminder` (interne) | Nécessaire ici pour appeler `IsOverdue` et `New` sans préfixe ? Non : tout est exporté, et `fakeMailer` n'a pas besoin du type `mailer` pour le satisfaire. Le test pourrait être externe — c'est l'exercice E1. |

## Erreurs fréquentes

1. **`time.Now()` au fond d'une fonction** de logique métier.
2. **`time.Sleep` dans un test.**
3. **Une interface de 10 méthodes** pour pouvoir mocker un client : le fake devient impossible.
4. **Définir l'interface côté fournisseur** « pour les tests ».
5. **Des mocks qui vérifient l'ordre et le nombre d'appels** : tests fragiles.
6. **Un fake qui ne simule jamais d'erreur.**
7. **Doubler ce qui est bon marché** (fichiers en `TempDir`, `bytes.Buffer`).
8. **Variable globale remplacée dans le test** (`var now = time.Now` modifiée) : tests non
   parallélisables et effets de bord entre tests.

## Bonnes pratiques Go

- D'abord un **paramètre** ; un champ si plusieurs méthodes en ont besoin ; une interface pour
  un comportement riche.
- Interfaces **côté consommateur**, petites, non exportées.
- Des **fakes** écrits à la main, réutilisables, capables de simuler les erreurs.
- Vérifier des **résultats**, pas des séquences d'appels.
- Garder réels les systèmes de fichiers temporaires et les buffers.
- Quelques tests d'intégration contre les vraies dépendances.

## Ce que je dois retenir

- Une dépendance cachée (`time.Now`, globale, client par défaut) rend le code intestable.
- Injecter par paramètre, champ, interface ou fonction.
- Stub (répond), fake (implémentation simplifiée), spy (enregistre), mock (vérifie les appels).
- Go préfère les fakes : petites interfaces, tests du résultat, pas de l'implémentation.
- Ne doubler que le lent, le non déterministe, l'externe.
- Jamais de `time.Sleep` dans un test.

---

➡️ [Exercices](exercices.md)
