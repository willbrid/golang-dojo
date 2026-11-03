# Projet 7 — `jobq`, une file de travaux fiable sur PostgreSQL

**À réaliser après le niveau 10.** Durée indicative : 20 à 30 heures.
Écrire le code dans `mes-solutions/projets/projet-07/`.

---

## Objectif pédagogique

Construire le composant que presque tout backend finit par écrire ou adopter : une **file de travaux en
arrière-plan** (envoyer des courriels, générer des rapports, appeler des services externes), persistante,
partagée par plusieurs processus, qui ne perd aucun travail et n'en exécute aucun deux fois **en effet**.
Le projet réunit la concurrence (niveau 6 : pool de workers, backpressure, arrêt gracieux, paniques), les
bases de données (niveau 8 : transactions, `FOR UPDATE SKIP LOCKED`, migrations) et l'architecture
(niveau 10 : *outbox*, idempotence, réessais avec *jitter*, disjoncteur, observabilité).

---

## Spécifications

Une bibliothèque `jobq` et un binaire de démonstration :

```go
q := jobq.New(db, jobq.Options{…})
q.Register("send_email", sendEmail)            // func(ctx context.Context, job jobq.Job) error
id, err := q.Enqueue(ctx, tx, "send_email", payload, jobq.RunAt(t), jobq.Key("bienvenue:42"))
err := q.Run(ctx)                               // workers, jusqu'à l'annulation de ctx
```

### Comportements exigés

| # | Exigence |
|---|---|
| S1 | **Enfilage transactionnel** : `Enqueue` accepte un `*sql.Tx` ; le travail n'existe que si la transaction de l'appelant est validée (patron *outbox*, N10 L8). Une variante sans transaction est fournie. |
| S2 | **Plusieurs processus**, chacun avec N workers (configurable), se partagent la file sans qu'un travail soit pris deux fois en même temps (`FOR UPDATE SKIP LOCKED`, ou bail — voir D1). |
| S3 | **Réessais** : un handler qui rend une erreur est réessayé avec attente exponentielle et *full jitter* (N10 L9), au plus `MaxAttempts` fois (par type de travail). Une erreur marquée permanente (`jobq.Permanent(err)`) n'est pas réessayée. |
| S4 | **Rebut** : après le dernier échec, le travail passe à l'état `dead`, avec la dernière erreur et l'historique des tentatives ; une commande permet de le relancer. |
| S5 | **Planification** : `RunAt(t)` diffère l'exécution ; un travail n'est jamais pris avant son heure. |
| S6 | **Unicité** : `Key(k)` empêche d'enfiler deux fois le même travail logique tant que le premier n'est pas terminé (contrainte en base). |
| S7 | **Délai d'exécution** : chaque type de travail a un délai ; un handler qui le dépasse est annulé (`context`), et la tentative compte comme un échec. |
| S8 | **Pannes de worker** : un processus tué (`SIGKILL`) au milieu d'un travail ne bloque pas ce travail indéfiniment : il est repris par un autre worker après un délai (D1). |
| S9 | **Panique** dans un handler : contenue, journalisée avec sa pile, comptée comme un échec (N6 L12). Le worker continue. |
| S10 | **Arrêt gracieux** : à l'annulation, plus aucun travail n'est pris ; ceux en cours ont `ShutdownTimeout` pour finir ; au-delà, ils sont annulés et **remis dans la file** (pas comptés comme des échecs). |
| S11 | **Backpressure** : `Enqueue` peut refuser (`ErrQueueFull`) au-delà d'un nombre configurable de travaux en attente par type ; la file ne grossit pas sans limite. |
| S12 | **Observabilité** : métriques par type (enfilés, réussis, échoués, au rebut, durée d'exécution en histogramme, âge du plus vieux travail en attente), journaux corrélés par identifiant de travail, et un `GET /jobs/stats` d'administration. |

---

## Contraintes

1. Bibliothèque standard, pgx (`database/sql` + `stdlib`, ou `pgxpool`), migrations embarquées. Aucun
   courtier externe : PostgreSQL **est** la file.
2. Le cœur (états, transitions, calcul des réessais) se teste **sans base**, avec horloge et aléa injectés.
3. Une suite de contrat pour le stockage, exécutée contre une implémentation en mémoire et contre
   PostgreSQL (N8 L7).
4. `go test -race ./...` passe ; aucun test ne laisse de goroutine (profil `goroutineleak` ou `synctest`
   là où c'est possible) ; aucun schéma de test ne reste en base.
5. `go vet`, `go fix -diff`, `golangci-lint` silencieux.
6. Un test de charge : 10 000 travaux, 3 processus × 8 workers, mesure du débit et de la latence (temps
   entre l'heure prévue et le début de l'exécution) au p50 et au p99.

---

## Difficultés glissées volontairement

- **D1. Verrou ou bail ?** Garder la transaction `FOR UPDATE` ouverte pendant toute l'exécution du
  handler (simple, mais une connexion et des verrous retenus pendant un travail de 30 s, N8 L4) — ou
  marquer le travail « pris jusqu'à t+délai » et valider tout de suite (un **bail**), avec reprise des baux
  expirés. Choisir, mesurer l'effet sur le pool de connexions, justifier.
- **D2.** Un travail pris, exécuté avec succès, mais le worker tombe avant d'enregistrer le succès. Il sera
  réexécuté. Que doit garantir le **handler** ? Comment `jobq` aide-t-il à écrire des handlers idempotents
  (identifiant de travail stable, numéro de tentative transmis) ?
- **D3.** Le *livelock* de N10 L8 : un lot de travaux pris ensemble, dont un échoue toujours. Votre
  conception l'évite-t-elle ? Le test qui le prouve.
- **D4.** L'horloge : plusieurs processus sur plusieurs machines dont les horloges diffèrent de quelques
  secondes. Qui décide qu'un travail est « dû » ou qu'un bail a expiré : l'horloge de Go ou `now()` de
  PostgreSQL ?
- **D5.** Le troupeau : 10 000 travaux planifiés à minuit pile. Que se passe-t-il à 00:00:00 ? Et quand une
  dépendance tombe et que 2 000 travaux échouent en même temps ?
- **D6.** L'équité : un type de travail très abondant (10 000 courriels) affame-t-il un type rare et urgent
  (une réinitialisation de mot de passe) ? Proposer des priorités ou des quotas par type.
- **D7.** La croissance de la table : des millions de travaux terminés. Purge par lots, index partiels,
  `VACUUM` — et l'effet de `SKIP LOCKED` sur une table qui contient surtout des lignes terminées.
- **D8.** Un handler appelle un service externe protégé par un disjoncteur (N10 L9). Le circuit est
  ouvert : faut-il compter un échec (et consommer une tentative), ou remettre le travail à plus tard sans
  pénalité ?

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.**

```
projet-07/
├── cmd/jobqdemo/main.go        configuration, migrations, Run, serveur d'administration
├── migrations/                 jobs, attempts (historique), index partiels
└── internal/
    ├── jobq/
    │   ├── job.go              Job, états, transitions (sans base)
    │   ├── retry.go            calcul des attentes (sans base)
    │   ├── queue.go            Enqueue, Register, Run (pool de workers)
    │   ├── worker.go           une boucle de worker : prendre, exécuter, enregistrer
    │   └── metrics.go
    ├── store/
    │   ├── store.go            interface : Enqueue, Claim, Complete, Fail, Requeue, Stats
    │   ├── memory/             pour tester le cœur et les workers
    │   ├── postgres/           SKIP LOCKED, baux, purge
    │   └── storetest/          suite de contrat
    └── admin/                  /jobs/stats, relance des travaux au rebut
```

L'idée directrice : le cœur décide (états, réessais), le stockage garantit (exclusivité, durabilité), les
workers orchestrent (concurrence, délais, arrêt). Chaque couche se teste seule.

---

## Tests à écrire

- Cœur : table des transitions d'état ; calcul des attentes (déterministe, aléa injecté).
- Contrat de stockage (mémoire et PostgreSQL) : prise exclusive sous concurrence, planification, unicité,
  reprise des baux expirés, rebut.
- Workers : réussite, échec puis réussite, échec permanent, panique, délai dépassé (horloge et attentes
  injectées ; `synctest` pour les délais si possible).
- Pannes : un processus « tué » (annulation brutale sans enregistrement) → le travail est repris (S8, D2).
- Arrêt gracieux : travaux en cours terminés ou remis en file ; aucun perdu (S10).
- Transaction : un `Enqueue` dans une transaction annulée ne produit aucun travail (S1).
- *Livelock* (D3) et troupeau (D5).
- Charge : 10 000 travaux, débit et latence p50/p99, aucun travail exécuté deux fois **en effet** (handler de
  test idempotent qui compte).

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 12 spécifications, démontrées par des tests et par le binaire de démonstration. |
| **C'est correct** | Les 8 difficultés traitées ou documentées ; aucun travail perdu ; exclusivité garantie ; `-race` silencieux ; aucune fuite de goroutine ni de connexion. |
| **C'est idiomatique** | Cœur sans base, interfaces côté consommateur, `context` partout, `select` pour les attentes, dépendances injectées, erreurs typées. |
| **Prêt pour la production** | Délais, backpressure, arrêt gracieux, purge, métriques exploitables (âge de la file), journaux corrélés, migrations sûres, débit mesuré. |

Rends ton code même incomplet : une conception discutée vaut mieux qu'un projet abandonné.
