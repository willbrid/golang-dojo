# Leçon 8 — Authentification

> L'authentification est le domaine où un code qui « marche » peut être catastrophiquement faux :
> un mot de passe haché avec SHA-256, un jeton comparé avec `==`, un message d'erreur qui dit « cet
> email n'existe pas », un cookie sans `HttpOnly`. Aucun test fonctionnel ne détecte ces fautes ;
> seuls des **principes** les évitent. Cette leçon les pose, avec la bibliothèque standard et
> `golang.org/x/crypto` — et la règle d'or : **ne jamais inventer sa propre cryptographie**.

## Objectifs

1. Stocker des mots de passe correctement (bcrypt ou argon2id), et dimensionner leur coût.
2. Gérer des sessions par jetons aléatoires et cookies sûrs.
3. Comprendre les jetons signés (JWT) et les clés d'API, et leurs compromis.
4. Éviter les fuites d'information (énumération, comparaisons non constantes).

## Explication

### Authentifier, autoriser

- **Authentification** : *qui* êtes-vous ? (mot de passe, jeton, certificat) — échec → **401**.
- **Autorisation** : avez-vous le *droit* ? (rôle, propriétaire de la ressource) — échec → **403** (ou
  404, leçon 5).

Cette leçon traite la première ; la seconde est une règle métier, vérifiée dans le service.

### Les mots de passe : un hachage **lent**

On ne stocke **jamais** un mot de passe, ni chiffré (une clé de déchiffrement peut fuir), ni haché avec
une fonction rapide (`sha256`). Une fonction de hachage rapide permet à un attaquant qui a volé la base
d'essayer des **milliards** de mots de passe par seconde sur une carte graphique.

On utilise une fonction de dérivation **volontairement lente** et **salée** :

| Algorithme | Où | Remarque |
|---|---|---|
| **argon2id** | `golang.org/x/crypto/argon2` | le choix recommandé aujourd'hui (coûteux en mémoire, résiste aux GPU) |
| **bcrypt** | `golang.org/x/crypto/bcrypt` | éprouvé, simple ; mots de passe limités à 72 octets |
| scrypt | `golang.org/x/crypto/scrypt` | coûteux en mémoire |
| PBKDF2 | `crypto/pbkdf2` (stdlib, Go 1.24) | quand une norme l'impose (FIPS) ; nombre d'itérations très élevé requis |

Le **sel** (une valeur aléatoire par mot de passe) empêche les tables précalculées et fait que deux
utilisateurs au même mot de passe ont des hachages différents. bcrypt le génère et le stocke **dans**
le hachage : on n'a rien à gérer.

Le **coût** est le paramètre de sécurité. Mesuré sur la machine de rédaction :

| Coût bcrypt | Une vérification |
|---|---|
| 4 (minimum, tests seulement) | 1,7 ms |
| 10 (défaut) | 62 ms |
| 12 | 247 ms |

Chaque +1 double le temps — pour vous **et** pour l'attaquant. On vise de l'ordre de 100 à 300 ms par
connexion, et on remonte le coût avec les années. Revers : la connexion devient une opération coûteuse,
qu'il faut **protéger contre les abus** (limite de débit par compte et par IP, leçon 3).

### Ne rien révéler

- **Un seul message** pour « email inconnu » et « mauvais mot de passe » : sinon l'attaquant découvre
  quels comptes existent (énumération).
- **Le même temps** dans les deux cas : vérifier un hachage factice quand l'email est inconnu, sinon la
  réponse instantanée trahit l'absence du compte.
- **Comparaisons en temps constant** pour les secrets (jetons, signatures) : `crypto/subtle.
  ConstantTimeCompare` ou `hmac.Equal`. Un `==` s'arrête au premier octet différent, et le temps de
  réponse révèle combien d'octets étaient justes.

### Les sessions

Après la connexion, le serveur crée une **session** : un jeton **aléatoire** (`crypto/rand`, 128 bits
au moins — `rand.Text()`), associé côté serveur à l'utilisateur et à une échéance. Le jeton est envoyé
dans un **cookie** :

| Attribut | Protège contre |
|---|---|
| `HttpOnly` | le vol du jeton par un script injecté (XSS) |
| `Secure` | l'envoi en clair sur HTTP |
| `SameSite=Lax` (ou `Strict`) | l'envoi du cookie depuis un autre site (CSRF) |
| `Path=/`, `MaxAge` | la portée et la durée |

Côté serveur, on stocke une **empreinte** du jeton (SHA-256) plutôt que le jeton lui-même : une fuite
de la table des sessions ne donne pas de jetons utilisables. (Un hachage rapide suffit ici : le jeton
est long et aléatoire, il n'y a rien à deviner par force brute.)

Avantages des sessions : révocables **immédiatement** (déconnexion, compte suspendu), jeton opaque,
rien de sensible chez le client. Inconvénient : un stockage partagé si plusieurs instances du service.

### Les jetons signés (JWT)

Un JWT contient les informations elles-mêmes (identifiant, rôles, échéance), **signées** par le
serveur. Il se vérifie **sans stockage** : pratique entre services, ou pour des clients non navigateurs.

Ses pièges sont nombreux et célèbres : algorithme `none` accepté, confusion entre algorithmes
(HMAC/RSA), clé faible, durée trop longue — et surtout, un JWT **ne se révoque pas** avant son échéance.
Règles : une bibliothèque éprouvée (`github.com/golang-jwt/jwt/v5`), un algorithme **imposé** à la
vérification, une durée courte (minutes) avec un jeton de rafraîchissement, et pas de JWT comme session
de navigateur si des sessions classiques suffisent.

### Les clés d'API

Pour les machines (scripts, intégrations) : une clé longue et aléatoire, transmise dans un en-tête
(`Authorization: Bearer …`), **stockée hachée** comme un jeton de session, avec un préfixe visible qui
permet de l'identifier dans les journaux sans la révéler (`sk_live_ab12…`), révocable et limitée en
droits.

### Déléguer : OAuth2 / OpenID Connect

« Se connecter avec… » repose sur OAuth2 et OpenID Connect : le fournisseur d'identité authentifie
l'utilisateur et votre service reçoit un jeton. `golang.org/x/oauth2` implémente le côté client. C'est
souvent le meilleur choix : pas de mots de passe à stocker du tout.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27

require golang.org/x/crypto v0.52.0
```

```go
// Fichier : auth/auth.go
// Package auth gère comptes, mots de passe et sessions.
package auth

import (
	"context"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"net/http"
	"sync"
	"time"

	"golang.org/x/crypto/bcrypt"
)

var (
	ErrInvalidCredentials = errors.New("identifiants invalides") // UN seul message : on ne dit pas lequel est faux
	ErrEmailTaken         = errors.New("adresse déjà utilisée")
	ErrWeakPassword       = errors.New("mot de passe trop court (12 caractères minimum)")
)

type session struct {
	email   string
	expires time.Time
}

type Service struct {
	mu       sync.Mutex
	users    map[string][]byte  // email → hash bcrypt ; jamais le mot de passe
	sessions map[string]session // empreinte SHA-256 du jeton → session ; jamais le jeton lui-même
	cost     int
	ttl      time.Duration
	now      func() time.Time
	dummy    []byte // hash factice, pour un temps de réponse identique si l'email est inconnu
}

func New(cost int, ttl time.Duration, now func() time.Time) *Service {
	dummy, _ := bcrypt.GenerateFromPassword([]byte("mot-de-passe-factice"), cost)
	return &Service{users: map[string][]byte{}, sessions: map[string]session{}, cost: cost, ttl: ttl, now: now, dummy: dummy}
}

func (s *Service) Register(email, password string) error {
	if len([]rune(password)) < 12 {
		return ErrWeakPassword
	}
	if len(password) > 72 { // bcrypt ignore tout au-delà de 72 octets : refuser plutôt que tronquer
		return errors.New("mot de passe trop long (72 octets maximum)")
	}
	hash, err := bcrypt.GenerateFromPassword([]byte(password), s.cost) // sel aléatoire inclus dans le hash
	if err != nil {
		return err
	}
	s.mu.Lock()
	defer s.mu.Unlock()
	if _, ok := s.users[email]; ok {
		return ErrEmailTaken
	}
	s.users[email] = hash
	return nil
}

func tokenKey(token string) string {
	sum := sha256.Sum256([]byte(token))
	return hex.EncodeToString(sum[:])
}

// Login vérifie le mot de passe et crée une session. Il rend le jeton à placer dans un cookie.
func (s *Service) Login(email, password string) (string, error) {
	s.mu.Lock()
	hash, ok := s.users[email]
	s.mu.Unlock()
	if !ok {
		hash = s.dummy // même travail que pour un compte existant : pas d'énumération par le temps
	}
	if err := bcrypt.CompareHashAndPassword(hash, []byte(password)); err != nil || !ok {
		return "", ErrInvalidCredentials
	}
	token := rand.Text() // 128 bits, crypto/rand (N4 L3)
	s.mu.Lock()
	s.sessions[tokenKey(token)] = session{email: email, expires: s.now().Add(s.ttl)}
	s.mu.Unlock()
	return token, nil
}

func (s *Service) Logout(token string) {
	s.mu.Lock()
	delete(s.sessions, tokenKey(token))
	s.mu.Unlock()
}

func (s *Service) userFor(token string) (string, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	sess, ok := s.sessions[tokenKey(token)]
	if !ok || !s.now().Before(sess.expires) {
		return "", false
	}
	return sess.email, true
}

// ---- HTTP ----

const cookieName = "session"

func SetSessionCookie(w http.ResponseWriter, token string, ttl time.Duration) {
	http.SetCookie(w, &http.Cookie{
		Name:     cookieName,
		Value:    token,
		Path:     "/",
		MaxAge:   int(ttl.Seconds()),
		HttpOnly: true,                 // inaccessible au JavaScript : un XSS ne vole pas la session
		Secure:   true,                 // envoyé seulement en HTTPS
		SameSite: http.SameSiteLaxMode, // pas envoyé sur les POST venant d'autres sites (CSRF)
	})
}

type userKey struct{}

// UserFrom rend l'utilisateur authentifié placé dans le contexte par RequireUser.
func UserFrom(ctx context.Context) (string, bool) {
	u, ok := ctx.Value(userKey{}).(string)
	return u, ok
}

// RequireUser est un middleware (leçon 3) : 401 sans session valide.
func (s *Service) RequireUser(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		c, err := r.Cookie(cookieName)
		if err != nil {
			http.Error(w, "authentification requise", http.StatusUnauthorized)
			return
		}
		email, ok := s.userFor(c.Value)
		if !ok {
			http.Error(w, "session invalide ou expirée", http.StatusUnauthorized)
			return
		}
		next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), userKey{}, email)))
	})
}
```

```go
// Fichier : auth/auth_test.go
package auth

import (
	"errors"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"golang.org/x/crypto/bcrypt"
)

func TestAuth(t *testing.T) {
	now := time.Date(2026, 11, 2, 9, 0, 0, 0, time.UTC)
	s := New(bcrypt.MinCost, time.Hour, func() time.Time { return now }) // coût minimal : tests rapides

	if err := s.Register("awa@x.fr", "court"); !errors.Is(err, ErrWeakPassword) {
		t.Errorf("mot de passe court : %v", err)
	}
	if err := s.Register("awa@x.fr", "correct horse battery"); err != nil {
		t.Fatal(err)
	}
	if err := s.Register("awa@x.fr", "autre mot de passe"); !errors.Is(err, ErrEmailTaken) {
		t.Errorf("doublon : %v", err)
	}
	if strings.Contains(string(s.users["awa@x.fr"]), "horse") {
		t.Fatal("le mot de passe est stocké en clair")
	}

	_, errWrongPw := s.Login("awa@x.fr", "mauvais mot de passe")
	_, errNoUser := s.Login("inconnu@x.fr", "correct horse battery")
	if !errors.Is(errWrongPw, ErrInvalidCredentials) || !errors.Is(errNoUser, ErrInvalidCredentials) {
		t.Errorf("les deux échecs doivent être indiscernables : %v / %v", errWrongPw, errNoUser)
	}

	token, err := s.Login("awa@x.fr", "correct horse battery")
	if err != nil {
		t.Fatal(err)
	}
	if _, stored := s.sessions[token]; stored {
		t.Error("le jeton brut ne doit pas être la clé de stockage")
	}

	protected := s.RequireUser(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		u, _ := UserFrom(r.Context())
		fmt.Fprint(w, "bonjour ", u)
	}))
	get := func(cookie string) (int, string) {
		req := httptest.NewRequest("GET", "/me", nil)
		if cookie != "" {
			req.AddCookie(&http.Cookie{Name: "session", Value: cookie})
		}
		rec := httptest.NewRecorder()
		protected.ServeHTTP(rec, req)
		return rec.Code, rec.Body.String()
	}
	if code, body := get(token); code != 200 || body != "bonjour awa@x.fr" {
		t.Errorf("session valide : %d %q", code, body)
	}
	if code, _ := get(""); code != 401 {
		t.Errorf("sans cookie : %d", code)
	}
	if code, _ := get("jeton-invente"); code != 401 {
		t.Errorf("jeton inventé : %d", code)
	}
	now = now.Add(2 * time.Hour) // la session (1 h) a expiré
	if code, _ := get(token); code != 401 {
		t.Errorf("session expirée : %d", code)
	}

	rec := httptest.NewRecorder()
	SetSessionCookie(rec, "abc", time.Hour)
	if c := rec.Header().Get("Set-Cookie"); !strings.Contains(c, "HttpOnly") || !strings.Contains(c, "Secure") || !strings.Contains(c, "SameSite=Lax") {
		t.Errorf("attributs de cookie manquants : %s", c)
	}
}

// Le coût de bcrypt, mesuré : c'est le paramètre de sécurité.
func BenchmarkBcrypt(b *testing.B) {
	for _, cost := range []int{bcrypt.MinCost, 10, 12} {
		hash, _ := bcrypt.GenerateFromPassword([]byte("correct horse battery"), cost)
		b.Run(fmt.Sprint("coût=", cost), func(b *testing.B) {
			for b.Loop() {
				bcrypt.CompareHashAndPassword(hash, []byte("correct horse battery"))
			}
		})
	}
}
```

`go get golang.org/x/crypto` puis `go test -race ./auth` : tout passe. Le benchmark
(`go test -run '^$' -bench . ./auth`) produit le tableau des coûts du cours.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `users` stocke un hachage bcrypt | Le test vérifie que le mot de passe n'apparaît pas en clair. |
| 12 caractères minimum, 72 octets maximum | La longueur compte plus que la complexité ; bcrypt ignorerait silencieusement la suite au-delà de 72 octets. |
| `ErrInvalidCredentials` pour les deux échecs | Indiscernables pour l'attaquant — le test le vérifie. |
| `s.dummy` comparé quand l'email est inconnu | Même durée de réponse : pas d'énumération par le temps. |
| `rand.Text()` | 128 bits d'aléa cryptographique. |
| Sessions indexées par `sha256(jeton)` | Le test vérifie que le jeton brut n'est pas la clé. |
| Expiration comparée à une horloge injectée | Le test fait avancer le temps de 2 h et constate le 401. |
| Cookie `HttpOnly`, `Secure`, `SameSite=Lax` | Trois attributs, trois attaques évitées — le test lit l'en-tête `Set-Cookie`. |
| `RequireUser` | Un middleware (leçon 3) : 401 sans session, utilisateur dans le contexte sinon. |
| `bcrypt.MinCost` dans les tests | Les tests restent rapides ; la production utilise 10 à 12. |

## Erreurs fréquentes

1. **Mot de passe en clair, chiffré, ou haché avec `sha256`/`md5`.**
2. **Un coût bcrypt de test en production.**
3. **Messages qui distinguent** « email inconnu » et « mot de passe faux ».
4. **`==` pour comparer des jetons ou des signatures.**
5. **`math/rand` pour un jeton** (N4 L3).
6. **Cookie sans `HttpOnly`/`Secure`/`SameSite`.**
7. **Jeton de session stocké tel quel** en base.
8. **JWT à longue durée sans révocation**, algorithme non imposé.
9. **Pas de limite de débit** sur la connexion : force brute en ligne.
10. **Écrire sa propre cryptographie.**

## Bonnes pratiques Go

- argon2id ou bcrypt (`golang.org/x/crypto`), coût mesuré sur la machine de production.
- Jetons par `crypto/rand`, stockés hachés, comparés en temps constant.
- Cookies `HttpOnly; Secure; SameSite`.
- Un message et un temps identiques pour tous les échecs de connexion.
- Limites de débit sur la connexion et l'inscription.
- Un middleware d'authentification qui place l'utilisateur dans le contexte.
- Déléguer à un fournisseur d'identité quand c'est possible.

## Ce que je dois retenir

- Authentification (401) ≠ autorisation (403).
- Mots de passe : hachage **lent et salé** (argon2id, bcrypt) ; le coût est un paramètre de sécurité.
- Rien ne doit révéler qu'un compte existe : même message, même temps.
- Secrets comparés en **temps constant**.
- Sessions : jeton aléatoire, stocké haché, cookie `HttpOnly` + `Secure` + `SameSite`.
- JWT : sans état mais non révocable ; bibliothèque éprouvée, algorithme imposé, durée courte.

---

➡️ [Exercices](exercices.md)
