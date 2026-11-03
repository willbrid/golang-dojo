# Projet 6 — `idp`, un service d'identité

**À réaliser après le niveau 8**, de préférence après le projet 4. Durée indicative : 20 à 30 heures.
Écrire le code dans `mes-solutions/projets/projet-06/`.

---

## Objectif pédagogique

L'authentification est le code où une erreur « mineure » devient une faille : un message d'erreur qui
révèle qu'un compte existe, un jeton de réinitialisation utilisable deux fois, une comparaison de
chaînes qui fuit par son temps d'exécution, une session qui survit au changement de mot de passe. Ce
projet construit un service d'identité **complet** : comptes, vérification d'email, sessions pour les
navigateurs, jetons pour les API, réinitialisation de mot de passe, clés d'API, journal d'audit. Il
mobilise N7 L5 (erreurs), N7 L8 (authentification), N7 L9 (production) et tout le niveau 8 (en
particulier les transactions et l'isolation, leçon 4).

---

## Spécifications

```
POST   /v1/accounts                       inscription (email non vérifié)          → 202
POST   /v1/accounts/verify                {token} vérifie l'email                   → 204
POST   /v1/sessions                       connexion navigateur : cookie de session  → 204
DELETE /v1/sessions/current               déconnexion                               → 204
DELETE /v1/sessions                       déconnexion de TOUTES les sessions         → 204
POST   /v1/tokens                         connexion API : jeton d'accès + jeton de rafraîchissement → 200
POST   /v1/tokens/refresh                 rotation du jeton de rafraîchissement      → 200
POST   /v1/password/forgot                {email} : envoie un lien (toujours 202)   → 202
POST   /v1/password/reset                 {token, password}                          → 204
POST   /v1/password/change                {current, new} (authentifié)               → 204
GET    /v1/me                             compte courant (cookie, jeton ou clé)     → 200
POST   /v1/api-keys                       crée une clé (affichée UNE fois)           → 201
GET    /v1/api-keys                       liste (préfixe, portée, dernière utilisation) → 200
DELETE /v1/api-keys/{id}                  révoque                                    → 204
GET    /v1/admin/audit?account=&cursor=   journal d'audit (rôle admin)               → 200
GET    /.well-known/jwks.json             clés publiques de vérification des jetons  → 200
```

### Comportements exigés

| # | Exigence |
|---|---|
| S1 | Mots de passe hachés (argon2id ou bcrypt, paramètres justifiés et configurables) ; longueur minimale 12, maximale 128 ; refus des 1 000 mots de passe les plus courants (liste embarquée). |
| S2 | **Aucune énumération de comptes** : inscription, connexion, « mot de passe oublié » rendent la même réponse, **et en un temps comparable**, que le compte existe ou non (mesurer). |
| S3 | Email vérifié par un jeton à usage unique valable 24 h ; la connexion est refusée tant que l'email n'est pas vérifié (message explicite, sans fuite). |
| S4 | Tout jeton secret (vérification, réinitialisation, rafraîchissement, session, clé d'API) est généré par `crypto/rand` (≥ 128 bits), et stocké **haché** (SHA-256) en base : une fuite de la base ne donne aucun jeton utilisable. |
| S5 | Sessions par cookie `HttpOnly`, `Secure`, `SameSite=Lax`, expiration glissante (30 min d'inactivité) et absolue (7 jours) ; identifiant de session **renouvelé** à la connexion. |
| S6 | Jetons d'accès signés (JWT, Ed25519), durée 10 min, algorithme **imposé** à la vérification, `kid` dans l'en-tête ; rotation des clés de signature sans invalider les jetons en cours (deux clés publiées pendant le recouvrement). |
| S7 | Jetons de rafraîchissement opaques, **à rotation** : chaque usage en délivre un nouveau et invalide l'ancien. La **réutilisation** d'un jeton déjà consommé révoque toute sa famille (signe de vol). |
| S8 | Réinitialisation : jeton à usage unique, 30 min ; utilisable **une seule fois même sous requêtes simultanées** ; après réinitialisation ou changement de mot de passe, **toutes** les sessions et familles de jetons sont révoquées. |
| S9 | Clés d'API : `idp_<préfixe>_<secret>` ; seul le préfixe est stocké en clair (pour l'affichage et la recherche), le secret haché ; portées (`read`, `write`) vérifiées par route ; date de dernière utilisation mise à jour sans écrire à chaque requête. |
| S10 | Limite de débit sur connexion, inscription et « oublié » : par compte **et** par IP, partagée entre instances (donc en base, ou justifier autrement). Pas de blocage de compte qui permettrait à un tiers de bloquer la victime. |
| S11 | Journal d'audit en base, écrit **dans la même transaction** que l'action : inscription, vérification, connexion réussie et échouée, réinitialisation, changement de mot de passe, création et révocation de clé, détection de réutilisation (S7). Jamais de secret ni de mot de passe dans l'audit ni dans les journaux. |
| S12 | Envoi d'emails derrière une interface ; en développement, une implémentation qui écrit dans une table `outbox` lisible par les tests. L'email n'est **jamais** envoyé dans la transaction (leçon 4). |

---

## Contraintes

1. Bibliothèque standard, pgx, `golang.org/x/crypto` ; `github.com/golang-jwt/jwt/v5` autorisé pour les JWT
   (ou ta propre implémentation Ed25519 minimale, justifiée et fuzzée).
2. PostgreSQL, migrations embarquées, *repositories* avec suite de contrat (niveau 8).
3. Horloge, générateur aléatoire et expéditeur d'emails **injectés** : les expirations se testent sans
   attendre.
4. Comparaisons de secrets en temps constant (`crypto/subtle`) là où c'est pertinent — et justifier où ça
   ne l'est pas (comparaison de hachés trouvés par index).
5. `go test -race ./...` passe ; couverture ≥ 85 % sur les paquets d'authentification ; fuzzing de
   l'analyse des jetons et des clés.
6. `go vet`, `go fix -diff`, `golangci-lint` silencieux.

---

## Difficultés glissées volontairement

- **D1.** Le temps de réponse de la connexion révèle-t-il l'existence du compte ? (Hacher un mot de passe
  coûte 100 ms ; ne rien hacher quand le compte n'existe pas coûte 1 ms.)
- **D2.** Deux requêtes simultanées utilisent le même jeton de réinitialisation. Avec « lire le jeton,
  vérifier `used_at IS NULL`, puis marquer utilisé », les deux réussissent-elles ? Écrire le test
  concurrent, puis la version en **une** instruction.
- **D3.** La rotation des jetons de rafraîchissement : un client mobile envoie deux rafraîchissements
  simultanés avec le même jeton (réseau instable, réessai). C'est une réutilisation — faut-il vraiment
  tout révoquer ? Proposer une fenêtre de tolérance et en discuter le risque.
- **D4.** Un JWT reste valide jusqu'à son échéance. L'utilisateur change son mot de passe : que deviennent
  ses jetons d'accès déjà émis ? Quelles options (durée courte, version de compte dans le jeton, liste de
  révocation), avec quels coûts ?
- **D5.** Le changement d'email : vérifier la **nouvelle** adresse avant de basculer, prévenir
  l'**ancienne**. Où stocker l'adresse en attente ?
- **D6.** La limite de débit par IP derrière un proxy (`X-Forwarded-For`, N7 L9) et derrière un NAT
  d'entreprise (mille employés, une IP).
- **D7.** Mettre à jour `last_used_at` d'une clé d'API à chaque requête = une écriture par requête de
  lecture. Comment l'éviter tout en restant à quelques minutes près ?
- **D8.** Le journal d'audit des connexions échouées sur des comptes **inexistants** : à qui
  l'attribuer, sans créer de faille d'énumération par le journal lui-même ?
- **D9.** Le format `idp_<préfixe>_<secret>` : pourquoi un préfixe reconnaissable aide-t-il les outils de
  détection de secrets dans les dépôts de code ?

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.**

```
projet-06/
├── cmd/idp/main.go
├── migrations/                  accounts, sessions, token_families, refresh_tokens,
│                                one_time_tokens, api_keys, audit, rate_limits, outbox
└── internal/
    ├── identity/                domaine : Account, règles de mot de passe, erreurs
    ├── secret/                  génération, hachage, comparaison constante, format des clés
    ├── jwtkeys/                 jeu de clés Ed25519, rotation, JWKS, signature, vérification
    ├── store/                   interfaces + postgres + memory + storetest
    ├── ratelimit/               fenêtres glissantes en base
    ├── mail/                    interface Sender, implémentation outbox
    └── httpapi/                 handlers, middlewares d'authentification (cookie, Bearer, clé)
```

L'idée directrice : `secret` et `jwtkeys` ne connaissent ni HTTP ni la base ; chaque règle de sécurité est
testable unitairement, et chaque scénario d'attaque a son test d'intégration.

---

## Tests à écrire

- Unitaires : force du mot de passe, génération et analyse des clés, hachage des jetons, signature et
  vérification des JWT (algorithme forcé à `none` ou `HS256` avec la clé publique → refusé ; `kid`
  inconnu → refusé ; expiré → refusé ; rotation).
- Fuzzing : analyse des clés d'API et des JWT ne panique jamais.
- Énumération (S2, D1) : mêmes statuts et mêmes corps ; écart de temps médian < 10 % sur 200 essais.
- Concurrence : D2 (réinitialisation), D3 (rafraîchissement), inscription simultanée du même email.
- Révocation : changement de mot de passe → anciennes sessions refusées, refresh refusé (S8).
- Expirations avec horloge injectée : vérification, réinitialisation, sessions (glissante et absolue).
- Audit : chaque action produit sa ligne, dans la même transaction (une action annulée n'en laisse pas).
- Aucun secret dans les journaux : un test capture la sortie du journal (un `io.Writer` injecté) et y cherche les jetons émis.

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 12 spécifications, les parcours complets (inscription → vérification → connexion → réinitialisation). |
| **C'est correct** | Les 9 difficultés traitées ou documentées ; aucun jeton réutilisable ; aucune énumération ; révocations effectives ; transactions correctes sous concurrence. |
| **C'est idiomatique** | Domaine sans HTTP, secrets encapsulés, interfaces côté consommateur, dépendances injectées, erreurs traduites à un seul endroit. |
| **Prêt pour la production** | Paramètres de hachage justifiés, rotation des clés, limites de débit partagées, audit exploitable, aucun secret journalisé. |

Rends ton code même incomplet : une conception discutée vaut mieux qu'un projet abandonné.
