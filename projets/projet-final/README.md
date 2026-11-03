# Projet final — `quorum`, un magasin clé-valeur répliqué

**À réaliser après le niveau 13.** Durée indicative : 60 à 90 heures.
Écrire le code dans `mes-solutions/projets/projet-final/`, dans un dépôt Git à lui.

---

## Objectif pédagogique

Construire un système dont la correction ne se voit pas en le lançant : un **magasin clé-valeur** réparti sur
plusieurs nœuds, qui continue de répondre quand un nœud tombe, et dont on peut **prouver** ce qu'il garantit.

Le service est petit (lire, écrire, supprimer une clé). Le travail est ailleurs :
- concevoir à partir d'invariants (module 1) ;
- tenir un budget d'allocation sur le chemin chaud (module 2) ;
- répliquer par quorums, avec des versions et des horloges, sous partitions (module 3) ;
- se donner les outils pour diagnostiquer ce qui ne se reproduit pas (modules 4 et 5) ;
- publier une API qu'on pourra faire évoluer (module 6) ;
- tenir la surcharge sans s'effondrer (module 7) ;
- documenter chaque décision importante, et la défendre en revue (module 8).

Le projet s'appuie aussi sur tout le reste du cours, en particulier :
- les niveaux 6 et 7 : concurrence et HTTP ;
- le niveau 10 : architecture et résilience ;
- le niveau 11 : mesure ;
- le niveau 12 : production.

Contrairement aux projets précédents, **les choix de conception te reviennent** : les spécifications disent ce
que le système garantit, pas comment. Chaque choix important fait l'objet d'un ADR.

---

## Spécifications

Un binaire `quorum`, lancé en N exemplaires (N = 3 ou 5), qui forment une grappe. Chaque nœud expose une API
HTTP JSON (ou gRPC, à justifier) :

```
PUT    /v1/kv/{key}      corps : la valeur (octets, 1 Mio au plus)   → 200 {"version": …}
GET    /v1/kv/{key}      → 200 la valeur, en-tête Quorum-Version     | 404
DELETE /v1/kv/{key}      → 200 {"version": …}
GET    /v1/kv/{key}?consistency=one      lecture sur un seul nœud, éventuellement périmée
```

### Comportements exigés

| # | Exigence |
|---|---|
| S1 | **Réplication** : chaque clé est stockée sur les N nœuds. Une écriture réussit quand W nœuds l'ont rendue durable ; une lecture interroge R nœuds. N, W et R sont configurables. La configuration par défaut doit garantir W + R > N (module 3). |
| S2 | **Versions** : chaque valeur porte une version, et deux écritures concurrentes de la même clé ne se perdent pas en silence. Choisir entre *last-writer-wins* avec une horloge hybride, des vecteurs de versions avec conflits exposés au client, ou une écriture conditionnelle (`If-Match`). Justifier par ADR (D1). |
| S3 | **Écriture conditionnelle** : `PUT` avec `If-Match: <version>` n'écrit que si la version courante est celle-là, sinon 412. C'est la brique d'un compteur ou d'un verrou côté client. |
| S4 | **Durabilité** : une écriture confirmée survit à un `kill -9` du nœud. Journal sur disque (*write-ahead log*) avec `fsync`, relu au démarrage, compacté périodiquement. Le coût du `fsync` est mesuré, et les écritures concurrentes partagent une validation (validation groupée, module 8). |
| S5 | **Réparation** : un nœud qui revient après une panne rattrape les écritures manquées, sans intervention. Par exemple : réparation à la lecture, et une comparaison périodique des répliques (arbres de hachage, ou résumé par plage de clés). |
| S6 | **Partitions** : une grappe coupée en deux continue de servir, du côté majoritaire, les opérations qui ont un quorum. Le côté minoritaire refuse les écritures (503), et sert les lectures `consistency=one` en le signalant. |
| S7 | **Contrôle d'admission** : file bornée, abandon des requêtes expirées, refus rapide (503 + `Retry-After`), délai propagé entre nœuds. Le client fourni (S11) a un budget de réessais (module 7). |
| S8 | **Budget du chemin chaud** : un `GET` local (`consistency=one`) fait au plus un nombre d'allocations fixé et mesuré (module 2). Le budget est vérifié par un test (`testing.AllocsPerRun`). |
| S9 | **Observabilité** (niveau 12) : journaux `slog` corrélés, métriques RED par opération, taille des files, retard de réplication par nœud, durée des `fsync`. Traces réparties entre nœuds. |
| S10 | **Boîte noire** (module 4) : enregistreur de vol, instantané automatique sur une requête lente, exposé sur l'adresse d'administration. |
| S11 | **Client Go** : un paquet `quorum/client`, avec une API pensée pour durer (module 6). Options extensibles, erreurs typées (`ErrConflict`, `ErrNoQuorum`), contexte partout. Documenté avec des exemples testés. |
| S12 | **Exploitation** (niveau 12) : configuration validée, sondes, arrêt gracieux qui ne perd aucune écriture confirmée, image *distroless*, `docker compose up` qui lance une grappe de trois nœuds. |

---

## Contraintes

1. Bibliothèque standard, `golang.org/x/sync`, `client_golang`, OpenTelemetry. Toute autre dépendance est
   justifiée par un ADR. Pas de bibliothèque de consensus ni de base de données embarquée : le journal et la
   réplication sont ton code.
2. `go test -race -shuffle=on ./...` passe. `govulncheck` et `staticcheck` sont propres.
3. **Une simulation déterministe** (module 3) : les nœuds tournent dans un seul processus, sur un réseau simulé
   qui retarde, perd, duplique et partitionne, en temps virtuel (`synctest`) et avec une graine. Le même code
   de nœud tourne en simulation et en vrai : le réseau et l'horloge sont des interfaces.
4. **Un vérificateur** (module 1) : l'historique des opérations d'un test (début, fin, résultat) est vérifié
   contre la garantie annoncée. C'est la linéarisabilité pour les opérations à quorum, si c'est ce que tu
   garantis ; sinon la garantie plus faible que tes ADR annoncent. 10 000 graines de simulation ne trouvent
   aucune violation, et une graine qui en trouvait une (pendant le développement) est gardée comme test.
5. **Au moins six ADR** dans `docs/adr/`, dont ceux des difficultés D1, D2, D4 et D6. Au moins un ADR est
   **remplacé** par un autre en cours de projet, quand une mesure a contredit la première décision.
6. **Mesures livrées**, chacune rejouable par un script :
   - un test par paliers en boucle ouverte, qui donne la capacité d'une grappe de 3 nœuds pour un objectif de
     p99 que tu fixes ;
   - le coût du `fsync`, avec et sans validation groupée ;
   - un test de pointe (module 7) qui montre que la grappe se rétablit seule ;
   - le comportement pendant la perte d'un nœud, puis pendant une partition : erreurs, latence, temps de
     rattrapage.

---

## Difficultés glissées volontairement

- **D1. Deux écritures concurrentes.** Deux clients écrivent la même clé au même instant, sur deux nœuds
  différents. Laquelle gagne, et qui le sait ?
  - Avec *last-writer-wins*, une horloge en avance fait gagner une écriture plus ancienne (module 3). Une
    horloge hybride réduit-elle le problème, ou le supprime-t-elle ?
  - Avec des vecteurs de versions, le client reçoit deux valeurs. Que fait-il ?
  - Écrire l'ADR, avec la garantie exacte qui en résulte.
- **D2. Ce que « durable » veut dire.** Le système d'exploitation peut garder les données en cache après un
  `write`. Que garantit `fsync` sur ta machine ? Que se passe-t-il si le disque est plein au milieu d'un
  enregistrement du journal ? Comment détecter un enregistrement tronqué au redémarrage ? Une somme de contrôle
  par enregistrement est un début.
- **D3. La suppression.** Un `DELETE` réussit sur W nœuds pendant qu'un troisième est en panne. Le nœud revient,
  avec l'ancienne valeur, et la réparation la recopie partout : la clé ressuscite. Il faut des pierres tombales
  (*tombstones*). Mais combien de temps les garder, et que se passe-t-il si un nœud revient après leur
  purge ?
- **D4. La lecture d'un quorum qui ne s'accorde pas.** Les R réponses diffèrent. Renvoyer la plus récente
  suffit-il à garantir qu'une lecture suivante ne verra pas une valeur plus ancienne ? Que doit faire la
  lecture (réparer avant de répondre ?) pour tenir la garantie annoncée ? Le vérificateur de la contrainte 4
  tranche.
- **D5. Le nœud lent.** Un nœud répond, mais en 2 s. Les écritures attendent-elles W réponses parmi les plus
  rapides, ou toujours les mêmes nœuds ? Combien de requêtes en cours vers chaque pair, au plus (une cloison
  par pair, N10 L9) ?
- **D6. Le changement de membres.** Ajouter un quatrième nœud à une grappe de trois est-il nécessaire dans la
  première version ? Si oui, comment éviter qu'une écriture soit confirmée par un quorum de l'ancienne
  configuration et invisible pour un quorum de la nouvelle ? Si non, l'écrire dans un ADR, avec ce que cela
  interdit.
- **D7. Le compactage.** Le journal grossit sans fin. Compacter pendant que des écritures arrivent, sans les
  bloquer et sans rien perdre si le nœud est tué au milieu du compactage.
- **D8. L'évolution du format.** Le format du journal sur disque et celui des messages entre nœuds changeront.
  Comment une grappe se met-elle à jour **nœud par nœud**, avec des nœuds de deux versions qui se parlent
  (module 6) ?

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.**

```
projet-final/
├── cmd/quorum/main.go          main → run(ctx, args, env, out) ; serve, inspect-wal, bench
├── docs/adr/                   0001-…md, 0002-…md
├── internal/
│   ├── node/                   la logique d'un nœud : coordination, quorums, réparation (sans réseau réel)
│   ├── version/                horloge hybride ou vecteurs de versions ; Compare
│   ├── wal/                    journal : enregistrements, sommes de contrôle, validation groupée, compactage
│   ├── store/                  la table en mémoire au-dessus du journal
│   ├── transport/              interface ; http/ (vrai réseau) ; sim/ (réseau simulé, graine)
│   ├── admission/              file bornée, refus, délais
│   ├── api/                    handlers HTTP publics, versionnés (/v1)
│   ├── obs/                    journaux, métriques, traces, boîte noire
│   └── lifecycle/              configuration, sondes, arrêt
├── client/                     le client Go public (S11)
├── check/                      le vérificateur d'historiques (contrainte 4)
├── sim/                        les scénarios de simulation, et la recherche de graines
├── Dockerfile, compose.yaml
└── scripts/                    les mesures de la contrainte 6
```

L'idée directrice : `node` ne connaît ni le réseau ni l'horloge réels. C'est ce qui permet de rejouer à
l'identique, en simulation, le scénario qui a cassé.

---

## Tests à écrire

- `version` : la comparaison est un ordre partiel correct (tests de propriétés, fuzzing).
- `wal` :
  - relecture après arrêt brutal à chaque octet possible d'un enregistrement (troncature simulée) ;
  - somme de contrôle invalide détectée ;
  - compactage interrompu.
- `node`, en simulation :
  - quorums atteints et non atteints ;
  - nœud lent ;
  - partition et cicatrisation ;
  - suppression et pierres tombales (D3) ;
  - 10 000 graines passées au vérificateur.
- `client` : erreurs typées, budget de réessais, exemples (`Example…`) qui tournent dans `go test`.
- `admission` : refus sous surcharge en temps virtuel, aucune requête traitée après son délai.
- Budget d'allocation du chemin chaud (S8).
- Intégration : trois vrais processus, un `kill -9` au milieu d'une charge, aucune écriture confirmée perdue.

---

## Livrables

1. Le dépôt : code, tests, `docker compose up` fonctionnel.
2. `docs/adr/` : au moins six ADR, dont un remplacé.
3. `docs/garanties.md` : une page qui dit **exactement** ce que le système garantit, et ce qu'il ne garantit
   pas. Chaque phrase renvoie à un test, à une mesure ou à un ADR.
4. `docs/mesures.md` : les résultats de la contrainte 6, avec la machine, la méthode et les commandes pour les
   refaire.
5. `docs/revue.md` : une revue de ton propre code, rédigée comme celle du module 8. Rédige-la après une semaine
   sans y toucher, avec ses commentaires classés : ce que tu corrigerais, ce que tu assumes, et pourquoi.

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 12 spécifications, démontrées par des tests, par la simulation et par `docker compose up`. |
| **C'est correct** | Le vérificateur ne trouve aucune violation sur 10 000 graines ; aucune écriture confirmée n'est perdue sur `kill -9` ; `-race` silencieux ; les 8 difficultés traitées ou documentées. |
| **C'est prouvé** | Chaque garantie de `docs/garanties.md` a sa preuve. Les mesures sont rejouables, et les chiffres annoncés sont ceux mesurés. |
| **C'est décidé** | Les ADR donnent contexte, critères, options mesurées, conséquences, réexamen. Le remplacement d'un ADR montre une décision corrigée par une mesure. |
| **C'est idiomatique** | Interfaces étroites (réseau, horloge, stockage), `context` partout, goroutines toutes arrêtées, erreurs typées, client public stable. |
| **Prêt pour la production** | Contrôle d'admission, arrêt gracieux, observabilité corrélée entre nœuds, boîte noire, image minimale. |

Un système qui garantit moins, mais le **prouve**, vaut mieux qu'un système qui promet tout sans preuve. Rends
ton travail même incomplet, avec `docs/garanties.md` à jour : c'est la règle du niveau 13 appliquée à ton
propre code.
