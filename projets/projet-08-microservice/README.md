# Projet 8 — `linkr`, un microservice prêt pour la production

**À réaliser après le niveau 12.** Durée indicative : 25 à 35 heures.
Écrire le code dans `mes-solutions/projets/projet-08/` (dans un dépôt Git à lui pour les parties CI).

---

## Objectif pédagogique

Le service lui-même est simple : un **raccourcisseur de liens** avec des statistiques de visites. Le travail
porte sur tout ce qui l'entoure, et qui distingue un programme qui marche d'un service qu'on peut confier à
une équipe d'exploitation :
- un dépôt et des versions propres ;
- un build reproductible et une image minimale ;
- une CI qui publie ;
- une configuration validée ;
- des journaux, métriques et traces corrélés ;
- un service durci, qui se déploie et s'arrête sans perdre une requête.

Le projet réunit tout le niveau 12, appuyé sur les niveaux 7 (HTTP), 8 (PostgreSQL), 10 (architecture,
résilience) et 11 (mesure).

---

## Spécifications

Une API JSON et une route de redirection :

```
POST   /api/links            {"url": "https://exemple.org/un/long/chemin"}  → 201 {"code": "k3Xq9", "short": "https://lnk.exemple/k3Xq9"}
GET    /api/links/{code}     → 200 {"url": …, "created_at": …, "visits": 1234}
DELETE /api/links/{code}     → 204 (réservé au propriétaire du lien)
GET    /{code}               → 302 Location: <url>     (le chemin chaud)
```

### Comportements exigés

| # | Exigence |
|---|---|
| S1 | **Création** : codes de 7 caractères tirés de `crypto/rand`, uniques (contrainte en base, nouvel essai en cas de collision). Chaque client authentifié par un jeton d'API (en-tête `Authorization: Bearer …`), vérifié en temps constant (leçon 8). |
| S2 | **Validation des URL** : schéma `http` ou `https` seulement, longueur bornée, hôte obligatoire. Refus des adresses privées, de bouclage et de métadonnées de nuage (`169.254.169.254`…), et du domaine du service lui-même (boucle). Voir D2. |
| S3 | **Redirection** : `GET /{code}` répond 302 en moins de 5 ms au p99 sous 2 000 requêtes/s sur un poste ordinaire, grâce à un cache en mémoire borné (N11 L6) devant PostgreSQL. Un code inconnu donne 404, un lien supprimé 410. |
| S4 | **Statistiques de visites** : comptées **sans** écriture en base sur le chemin chaud. Elles sont agrégées en mémoire et écrites par lots toutes les N secondes, et lors de l'arrêt. Perte maximale acceptée en cas de `SIGKILL` : N secondes de visites, documentée. |
| S5 | **Limitation de débit** par jeton d'API pour la création (N6 L4), et une limite globale de requêtes en cours (N6 L11) : 429 ou 503 avec `Retry-After`, jamais une file qui grossit. |
| S6 | **Configuration** (leçon 5) : préfixe `LINKR_`, précédence défauts < fichier < environnement < options, validation complète au démarrage (code de sortie 2), secrets par `*_FILE` (URL de la base, jetons d'API). |
| S7 | **Journaux** (leçon 6) : `slog` JSON, une ligne par requête, `trace_id` dans chaque ligne. Aucune URL complète de lien ni aucun jeton dans les journaux : seulement le code et l'identifiant du client. |
| S8 | **Métriques** (leçon 7) : RED par route (le motif), taux de succès du cache, taille de l'agrégat de visites en attente, âge du dernier lot écrit, plus les collecteurs Go et processus, sur l'adresse d'administration. |
| S9 | **Traces** (leçon 7) : `otelhttp` en entrée, spans autour des accès à la base et au cache, export OTLP configurable, `ParentBased(TraceIDRatioBased)`. |
| S10 | **Sondes et arrêt** (leçon 9) : `/livez`, `/readyz` (prête après migration et chargement), et sur `SIGTERM` la séquence : plus prêt, attente, `Shutdown` borné, écriture du dernier lot de visites, fermeture de la base et des traces. |
| S11 | **Version** (leçons 1 et 2) : `-version` et `GET /version` (adresse d'administration) donnent version, commit, date et état `dirty`. Un binaire `+dirty` ou `(devel)` refuse de démarrer si `LINKR_ENV=production`. |
| S12 | **Livraison** (leçons 2 à 4) : `ci.sh`, un workflow CI et un workflow de publication. Ils produisent des binaires reproductibles pour 3 plateformes avec `SHA256SUMS` signé, et une image *distroless* `nonroot` multi-architecture. Le tout validé par `actionlint`, et exécuté au moins une fois dans ton dépôt de test. |

---

## Contraintes

1. Bibliothèque standard, `pgx`, `client_golang`, OpenTelemetry, `golang.org/x/sync`. Toute autre dépendance
   est justifiée par écrit (leçon 8).
2. Architecture du niveau 10 : domaine sans dépendance technique, dépôt derrière une interface côté
   consommateur, suite de contrat contre une implémentation en mémoire et contre PostgreSQL.
3. `go test -race -shuffle=on ./...` passe. Les tests d'intégration sont derrière `//go:build integration`,
   et ils échouent, au lieu de se sauter, si la base manque.
4. `ci.sh` passe en local et en CI : `tidy`, `gofmt`, `vet` (avec les étiquettes), `staticcheck`, tests,
   `govulncheck`, build.
5. `docker compose up` lance le service, PostgreSQL, Prometheus et Jaeger. Les migrations passent par un
   service `migrate` qui se termine avant l'API.
6. Mesures livrées (niveau 11) :
   - le p50 et le p99 de `GET /{code}` sous charge en boucle **ouverte** (N11 L7) ;
   - le coût de l'instrumentation par requête ;
   - le nombre d'erreurs pendant un déploiement progressif de deux instances (leçon 9), avec la méthode
     utilisée pour le mesurer.

---

## Difficultés glissées volontairement

- **D1. Le cache et la suppression.** Un lien supprimé doit répondre 410 **partout**, alors que chaque
  instance a son propre cache. Durée de vie courte ? Invalidation par `LISTEN/NOTIFY` de PostgreSQL ?
  Accepter une fenêtre d'incohérence ? Chiffrer cette fenêtre et la justifier.
- **D2. La redirection ouverte et le SSRF.** Le service lui-même ne télécharge jamais l'URL. Mais si une
  fonction « aperçu du lien » était ajoutée, elle le ferait. La validation de S2 suffit-elle ?

  Un nom de domaine qui **résout** vers `127.0.0.1`, et une redirection en cascade, sont des pistes à
  examiner (N7 L9). Écrire les tests de refus et dire ce que S2 protège, et ce qu'il ne protège pas.
- **D3. Les visites perdues.** Entre deux lots, les visites sont en mémoire. Que perd-on sur un `SIGKILL` ?
  Sur un déploiement propre ? Et si l'écriture du lot échoue (base indisponible) : réessayer, garder en
  mémoire (jusqu'à quelle taille ?), ou abandonner et le compter dans une métrique ?
- **D4. L'énumération.** Des codes de 7 caractères sur 62 symboles : combien de codes possibles ? Un
  attaquant qui essaie 1 000 codes par seconde trouve-t-il des liens privés ? Faut-il limiter le débit des
  404 ? Les journaliser ?
- **D5. Le lien très populaire.** Un code reçoit 50 % du trafic. Les statistiques par code dans l'agrégat
  deviennent-elles un point de contention (N11 L5–L6) ? Le mesurer et, si besoin, le corriger (compteurs
  répartis).
- **D6. Le démarrage à froid.** Une nouvelle instance a un cache vide : son p99 est mauvais pendant les
  premières secondes. `/readyz` doit-il attendre un préchargement ? Lequel, et combien de temps ?
- **D7. La cardinalité.** Les métriques par code de lien seraient tentantes (« visites de `k3Xq9` »). Pourquoi
  non, et où cette information doit-elle vivre ?
- **D8. La rotation des jetons d'API.** Un client change son jeton sans interruption (leçon 8). Comment le
  service accepte-t-il les deux pendant la transition, avec des jetons lus dans un fichier qui change sous
  lui ?

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.**

```
projet-08/
├── cmd/linkr/main.go           main → run(ctx, args, env, out) ; sous-commandes serve, migrate, healthcheck
├── migrations/                 links, visits (agrégées), index
├── internal/
│   ├── link/                   le domaine : Link, codes, validation des URL (sans dépendance technique)
│   ├── store/                  interface ; memory/ ; postgres/ ; storetest/ (contrat)
│   ├── cache/                  cache borné (LRU ou par segments), invalidation
│   ├── visits/                 agrégat en mémoire, écriture par lots, arrêt
│   ├── api/                    handlers, authentification, limitation de débit
│   ├── config/                 leçon 5
│   ├── obs/                    logger, métriques, traces (leçons 6 et 7)
│   ├── buildinfo/              leçon 1
│   └── lifecycle/              démarrage, sondes, arrêt (leçon 9)
├── Dockerfile, .dockerignore, compose.yaml, prometheus.yml, alerts.yml
├── ci.sh, build.sh
└── .github/workflows/          ci.yml, release.yml
```

L'idée directrice : le domaine ne sait rien de HTTP ni de PostgreSQL. `run` assemble tout, et chaque
préoccupation de production (configuration, télémétrie, cycle de vie) est un paquet testé seul, branché une
fois.

---

## Tests à écrire

- Domaine : génération et validation des codes ; validation des URL par table, cas de refus de D2 compris.
- Contrat du dépôt (mémoire et PostgreSQL) : création, unicité, suppression, visites agrégées.
- Cache : borne respectée, invalidation (D1), comportement sous `-race`.
- Visites : agrégation concurrente, lot écrit à l'arrêt, échec d'écriture (D3).
- API : authentification (jeton absent, faux, ancien et nouveau pendant une rotation), limitation de débit,
  404 et 410, aucune donnée sensible dans les journaux produits (relus en JSON, leçon 6).
- Configuration : précédence, validation, secrets par fichier, refus de démarrer en production avec un
  binaire `+dirty`.
- Cycle de vie : séquence d'arrêt dans l'ordre, dans le budget, `readyz` à 503 pendant la propagation.
- Charge et déploiement : les mesures de la contrainte 6, rejouables par un script.

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 12 spécifications, démontrées par des tests, par `docker compose up` et par une publication réelle. |
| **C'est correct** | Les 8 difficultés traitées ou documentées avec des chiffres ; aucune requête perdue pendant un déploiement propre ; `-race` silencieux ; aucune donnée sensible dans les journaux, métriques ou traces. |
| **C'est idiomatique** | Domaine pur, interfaces côté consommateur, `run` testable, `context` partout, configuration et télémétrie injectées, erreurs typées. |
| **Prêt pour la production** | Build reproductible et signé, image minimale non privilégiée, CI aux droits minimaux, sondes justes, arrêt borné, alertes sur les symptômes, mesures de performance et de déploiement fournies. |

Rends ton code même incomplet : une conception discutée vaut mieux qu'un projet abandonné.
