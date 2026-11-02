# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Pourquoi pas SHA-256 ?

Ordres de grandeur : un `sha256.Sum256` d'un mot de passe court coûte **quelques centaines de
nanosecondes** sur un cœur (plusieurs millions d'essais par seconde) ; une vérification bcrypt au coût 10,
**~60 ms** (mesuré dans le cours) : ~16 essais par seconde. Facteur : plusieurs centaines de milliers.

Sur 10 000 cœurs : des dizaines de milliards de SHA-256 par seconde (et bien plus sur GPU, très efficaces
pour SHA-256) — un mot de passe de 8 caractères tombe en heures — contre ~160 000 bcrypt par seconde,
qui laissent des années pour les mêmes mots de passe. C'est toute la raison d'être d'un hachage lent.

## E2 — Le sel

Les deux hachages sont **différents**, et les deux vérifications réussissent. Format :
`$2a$10$<22 caractères de sel><31 caractères de hachage>` — `2a` est la variante, `10` le coût, puis le sel
et le hachage en base64 (variante bcrypt). Le sel est **dans** le hachage stocké : `CompareHashAndPassword`
le relit pour recalculer.

## E3 — L'énumération par le temps

Sans hachage factice : un email inconnu répond en quelques **microsecondes**, un email existant en
**~60 ms** (coût 10). L'écart est énorme, visible dès la première mesure — un script énumère les comptes
existants sans jamais connaître de mot de passe. Avec le hachage factice : les deux distributions de temps
se confondent.

## E4 — Comparaison en temps constant

`==` compare octet par octet et s'arrête à la première différence. L'écart par octet est minuscule, mais
un attaquant peut faire des **millions** de mesures et utiliser des statistiques : le bruit se moyenne,
l'écart apparaît, et il devine la clé **octet par octet** (256 essais par octet au lieu de 256^32 pour une
clé de 32 octets). `subtle.ConstantTimeCompare` examine toujours tous les octets.

Il rend 0 immédiatement si les **longueurs** diffèrent, ce qui révèle la longueur. Comparer les
**empreintes SHA-256** des deux valeurs donne toujours 32 octets des deux côtés : la longueur ne fuit plus,
et l'on peut stocker l'empreinte plutôt que la clé.

## E5 — Les attributs du cookie

- `HttpOnly` : un script injecté dans la page (XSS) ne peut pas lire `document.cookie` et envoyer la
  session à l'attaquant.
- `Secure` : sur un Wi-Fi public, une requête HTTP en clair (lien `http://` cliqué par erreur) ne
  transporte pas le cookie, que n'importe qui sur le réseau pourrait lire.
- `SameSite=Lax` : un formulaire `POST` sur un site malveillant, qui cible votre service, n'emporte pas le
  cookie de session — la requête arrive non authentifiée (CSRF).

`Strict` : quand même une **navigation** depuis un autre site (un lien `GET` cliqué dans un courriel) ne
doit pas arriver connectée — banque, administration. Inconvénient : l'utilisateur qui arrive par un lien
semble déconnecté.

## Exercice intermédiaire — Connexion complète

Points clés :

- **CSRF** : `SameSite=Lax` protège les `POST` inter-sites dans les navigateurs modernes, mais pas les
  `GET` qui modifieraient l'état (à proscrire de toute façon), ni les anciens navigateurs, ni certaines
  configurations de sous-domaines. La défense en profondeur vérifie l'origine de la requête.
  `http.CrossOriginProtection` (Go 1.25) le fait pour les méthodes non sûres en s'appuyant sur l'en-tête
  `Sec-Fetch-Site` (et `Origin` en repli), sans jeton :

  ```go
  cop := http.NewCrossOriginProtection()
  handler := cop.Handler(mux)       // rejette les POST/PUT/DELETE venus d'une autre origine
  ```

- **Limites de débit** : par email **et** par IP. Par email seulement, un attaquant teste un mot de passe
  courant sur des milliers de comptes (*password spraying*) ; par IP seulement, il répartit sur un réseau
  de machines.
- **Logout** : supprimer la session **côté serveur** (sinon le jeton reste valide) et envoyer un cookie
  expiré (`MaxAge: -1`).
- **Tests** : un `http.Client{Jar: jar}` avec `cookiejar.New(nil)` conserve le cookie entre `login` et
  `me`, exactement comme un navigateur. `Secure: true` empêche le cookie d'être envoyé sur `http://` :
  en test, utiliser `httptest.NewTLSServer` et son `srv.Client()`, ou rendre `Secure` configurable.

## Défi — Un jeton signé, pour comprendre

```go
func Sign(c Claims, key []byte) string {
	payload, _ := json.Marshal(c)
	p := base64.RawURLEncoding.EncodeToString(payload)
	mac := hmac.New(sha256.New, key)
	mac.Write([]byte(p))
	return p + "." + base64.RawURLEncoding.EncodeToString(mac.Sum(nil))
}

func Verify(token string, key []byte, now time.Time) (Claims, error) {
	p, sig, ok := strings.Cut(token, ".")
	if !ok {
		return Claims{}, ErrMalformed
	}
	got, err := base64.RawURLEncoding.DecodeString(sig)
	if err != nil {
		return Claims{}, ErrMalformed
	}
	mac := hmac.New(sha256.New, key)
	mac.Write([]byte(p))
	if !hmac.Equal(got, mac.Sum(nil)) {          // temps constant
		return Claims{}, ErrBadSignature
	}
	payload, err := base64.RawURLEncoding.DecodeString(p)   // décoder APRÈS avoir vérifié
	if err != nil {
		return Claims{}, ErrMalformed
	}
	var c Claims
	if err := json.Unmarshal(payload, &c); err != nil {
		return Claims{}, ErrMalformed
	}
	if !now.Before(time.Unix(c.Exp, 0)) {
		return Claims{}, ErrExpired
	}
	return c, nil
}
```

Ce qu'une vraie bibliothèque gère en plus : l'**en-tête** avec l'algorithme et sa vérification stricte
(l'attaque `alg: none`, la confusion HMAC/RSA) ; les algorithmes asymétriques (signer avec une clé privée,
vérifier avec une clé publique distribuée — indispensable entre services) ; la **rotation** des clés
(`kid`) ; les champs standard (`nbf`, `iat`, `aud`, `iss`) et la tolérance d'horloge ; l'interopérabilité
avec les autres langages. Une erreur dans l'un de ces points ouvre une faille d'authentification : c'est
exactement le code qu'on ne doit pas écrire soi-même.

## Quiz

1. Authentification : qui êtes-vous (401) ; autorisation : en avez-vous le droit (403).
2. Il permet des milliards d'essais par seconde à un attaquant qui a volé la base.
3. À rendre uniques les hachages de mots de passe identiques et inutiles les tables précalculées ; dans
   la chaîne du hachage elle-même.
4. En mesurant sur la machine de production : viser ~100 à 300 ms par vérification.
5. Pour ne pas révéler quels comptes existent ; non, il faut aussi un temps de réponse identique.
6. Pour que le temps de réponse ne révèle pas combien d'octets sont justes.
7. Le vol par script (XSS), l'envoi en clair, l'envoi depuis un autre site (CSRF).
8. Une fuite de la table ne fournit alors aucun jeton utilisable.
9. Il ne se révoque pas avant son échéance.
10. Hachée (comme un jeton de session), avec un préfixe visible pour l'identifier, révocable et limitée.
