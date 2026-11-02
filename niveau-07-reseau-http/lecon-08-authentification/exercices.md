# Leçon 8 — Exercices

Code dans `mes-solutions/niveau-07/lecon-08/`. `go get golang.org/x/crypto`.

## Exercices faciles

**E1 — Pourquoi pas SHA-256 ?**
Mesurer avec un benchmark le temps d'un `sha256.Sum256` d'un mot de passe, et d'un
`bcrypt.CompareHashAndPassword` au coût 10. Combien de mots de passe par seconde un attaquant
testerait-il avec chacun, sur **un** cœur ? Avec 10 000 cœurs (une ferme de GPU, grossièrement) ?

**E2 — Le sel.**
Hacher deux fois le même mot de passe avec bcrypt. Les deux hachages sont-ils identiques ? Les deux
vérifications réussissent-elles ? Où est le sel dans la chaîne produite ? *(Décomposer le format
`$2a$10$…`.)*

**E3 — L'énumération par le temps.**
Retirer le hachage factice de `Login`. Mesurer le temps de réponse pour un email existant (mauvais mot de
passe) et pour un email inconnu, 100 fois chacun. L'écart est-il visible ? Remettre le hachage factice et
remesurer.

**E4 — Comparaison en temps constant.**
Écrire une vérification de clé d'API avec `==`, puis avec `subtle.ConstantTimeCompare`. Pourquoi la
seconde est-elle nécessaire même si l'écart de temps est de quelques nanosecondes ? *(Penser au nombre de
mesures qu'un attaquant peut faire.)* Pourquoi comparer des **empreintes** (SHA-256 des deux côtés)
résout-il aussi le problème des longueurs différentes ?

**E5 — Les attributs du cookie.**
Pour chacun des attributs `HttpOnly`, `Secure`, `SameSite`, décrire l'attaque concrète qu'il empêche,
en une phrase chacune. Dans quel cas faut-il `SameSite=Strict` plutôt que `Lax` ?

---

## Exercice intermédiaire — Connexion complète

Brancher le service `auth` de l'exemple sur un vrai serveur HTTP :

1. Routes : `POST /v1/register`, `POST /v1/login` (pose le cookie), `POST /v1/logout` (supprime la
   session et le cookie), `GET /v1/me` (protégée).
2. Limite de débit sur `/v1/login` : 5 tentatives par minute **par email** et 20 **par IP** (leçon 3) ;
   au-delà, 429.
3. Toutes les erreurs au format *problem details* (leçon 5) ; aucune ne révèle si l'email existe.
4. Une page de connexion HTML servie avec `html/template` (N4 L8). Les formulaires `POST` qui modifient
   l'état sont protégés contre le CSRF : `SameSite=Lax` suffit-il ? Ajouter la vérification de l'en-tête
   `Origin` sur les requêtes qui modifient l'état. *(Go 1.25 propose `http.CrossOriginProtection` :
   lire sa documentation et l'utiliser.)*
5. Tests `httptest` du parcours complet avec un `http.Client` muni d'un `cookiejar` (`net/http/cookiejar`).
6. Le coût bcrypt vient de la configuration ; les tests utilisent `bcrypt.MinCost`.

---

## Défi — Un jeton signé, pour comprendre

Sans bibliothèque JWT (**pour l'exercice uniquement**) :

1. Écrire `Sign(claims Claims, key []byte) string` qui encode `{"sub":…,"exp":…}` en base64url et ajoute
   une signature HMAC-SHA256 (`crypto/hmac`).
2. Écrire `Verify(token string, key []byte, now time.Time) (Claims, error)` : signature vérifiée avec
   `hmac.Equal`, échéance vérifiée, format strict.
3. Fuzzer `Verify` (N5 L7) : aucune entrée ne doit le faire paniquer, et aucune entrée non signée par la
   clé ne doit être acceptée.
4. Lister trois choses qu'une vraie bibliothèque JWT gère et que votre version ignore. Conclure : pourquoi
   ne jamais utiliser cette version en production ?

---

## Quiz

1. Différence entre authentification et autorisation ? Quels statuts ?
2. Pourquoi un hachage rapide est-il inadapté aux mots de passe ?
3. À quoi sert le sel ? Où bcrypt le stocke-t-il ?
4. Comment choisir le coût de bcrypt ?
5. Pourquoi le même message pour « email inconnu » et « mauvais mot de passe » ? Est-ce suffisant ?
6. Pourquoi comparer des secrets en temps constant ?
7. Que protègent `HttpOnly`, `Secure` et `SameSite` ?
8. Pourquoi stocker l'empreinte d'un jeton de session plutôt que le jeton ?
9. Quel est le principal inconvénient d'un JWT par rapport à une session ?
10. Comment stocker une clé d'API ?
