# Projet 5 — `kvd`, un serveur clé-valeur concurrent

**À réaliser après le niveau 6.** Durée indicative : 12 à 20 heures.
Écrire le code dans `mes-solutions/projets/projet-05/`.

---

## Objectif pédagogique

Écrire un vrai serveur réseau, utilisé par de nombreux clients **en même temps**, et le faire
**correctement** : sans *data race*, sans fuite de goroutines, avec des délais, une file bornée et un
**arrêt gracieux**. Tout le niveau 6 sert ici : goroutines, channels, `select`, verrous, `context`,
patrons, pathologies, backpressure, frontières de panique.

> **Anticipation signalée.** Le réseau TCP est traité en détail au niveau 7 (leçon 10). Ce projet
> n'en utilise que trois éléments, qui reposent entièrement sur ce que tu connais déjà :
>
> ```go
> ln, err := net.Listen("tcp", "127.0.0.1:7070")   // écouter
> conn, err := ln.Accept()                           // attendre un client (bloque)
> // conn est un net.Conn : un io.Reader + io.Writer + Close() (N4 L5)
> conn.SetDeadline(time.Now().Add(30 * time.Second)) // délai sur les lectures/écritures
> ```
>
> `ln.Close()` débloque un `Accept` en cours (qui rend alors une erreur `net.ErrClosed`), et
> `conn.Close()` débloque un `Read` en cours : c'est ainsi qu'on arrête un serveur. Pour tester à la
> main : `nc 127.0.0.1 7070` (ou `telnet`).

---

## Spécifications

Un protocole texte, une commande par ligne, une réponse par ligne :

```
$ nc 127.0.0.1 7070
SET langage go
OK
GET langage
VALUE go
GET absent
NOT_FOUND
INCR visites
VALUE 1
DEL langage
OK
KEYS lang*
KEYS
SETEX session 10 abc123
OK
STATS
STATS clients=3 keys=2 commands=128 rejected=0
QUIT
BYE
```

### Comportements exigés

| # | Exigence |
|---|---|
| S1 | Commandes : `SET k v`, `GET k`, `DEL k`, `INCR k`, `SETEX k secondes v`, `KEYS motif`, `STATS`, `QUIT`. La valeur de `SET` est **tout le reste de la ligne** (elle peut contenir des espaces). |
| S2 | Une commande inconnue ou mal formée rend `ERR <message>` et la connexion **reste ouverte**. |
| S3 | Chaque client est servi dans sa propre goroutine ; des centaines de clients simultanés sont servis sans qu'un client lent ralentisse les autres. |
| S4 | `INCR` sur une valeur non entière rend `ERR` ; `INCR` est **atomique** : 100 clients qui font 1 000 `INCR` chacun donnent exactement 100 000. |
| S5 | `SETEX` : la clé expire après le délai ; une clé expirée n'est jamais rendue. Une goroutine de nettoyage purge périodiquement les clés expirées, et s'arrête avec le serveur. |
| S6 | `KEYS motif` accepte `*` et `?` (indice : `path.Match`) ; résultat **trié**. |
| S7 | **Délai d'inactivité** : une connexion sans commande pendant 30 s (configurable) est fermée. |
| S8 | **Limite de clients** : au-delà de N connexions simultanées (configurable, défaut 100), une nouvelle connexion reçoit `ERR serveur plein` et est fermée immédiatement. |
| S9 | **Ligne trop longue** (> 64 Kio) : `ERR ligne trop longue`, puis fermeture de la connexion. |
| S10 | **Arrêt gracieux** sur `SIGINT`/`SIGTERM` : on cesse d'accepter, on laisse les commandes **en cours** se terminer (au plus 5 s), on ferme toutes les connexions, puis le programme se termine avec le code 0. |
| S11 | Une panique pendant le traitement d'une commande ne fait tomber **que** la connexion concernée (journalisée avec sa pile). |
| S12 | `STATS` donne le nombre de clients connectés, de clés, de commandes traitées et de connexions refusées. |

---

## Contraintes

1. **Bibliothèque standard uniquement** (plus `golang.org/x/sync` si tu le souhaites).
2. Le **stockage** est un paquet `internal/store` **indépendant du réseau** : il se teste sans aucun
   socket. Il reçoit son horloge (`now func() time.Time`, N5 L3).
3. Le **protocole** (analyse d'une ligne en commande) est un paquet `internal/protocol`, testé par une
   table et **fuzzé** (N5 L7) : aucune entrée ne doit le faire paniquer.
4. Le **serveur** (`internal/server`) expose `func (s *Server) Serve(ctx context.Context, ln net.Listener) error`
   : il s'arrête quand `ctx` est annulé. `main` ne fait que créer le listener, le contexte
   (`signal.NotifyContext`) et appeler `Serve`.
5. `go test -race ./...` passe, et **aucun test ne laisse de goroutine** derrière lui (vérifier avec le
   profil `goroutineleak` ou `synctest` là où c'est possible).
6. Aucune variable globale mutable.
7. `go vet` silencieux, `go fix -diff` vide.

---

## Difficultés glissées volontairement

- **D1.** `Accept` bloque. Comment la boucle d'acceptation voit-elle l'annulation du contexte ?
  (Relire l'encadré : qui doit fermer le listener, et quand ?)
- **D2.** Une goroutine de connexion bloquée dans `Read` (client silencieux) ne voit pas `ctx.Done()`.
  Comment l'arrêter pendant l'arrêt gracieux ? (Même mécanisme, côté `conn`.)
- **D3.** Lister les connexions ouvertes pour les fermer à l'arrêt suppose une structure **partagée**
  entre la boucle d'acceptation et les goroutines de connexion. Laquelle, protégée comment ? Et qui
  retire une connexion de la structure quand elle se termine d'elle-même ?
- **D4.** `INCR` sur une clé avec expiration (`SETEX`) : l'expiration est-elle conservée ? Décider,
  documenter, tester.
- **D5.** Un client qui envoie `SET k v` sans jamais lire les réponses : le serveur bloque-t-il sur
  `Write` ? Pour combien de temps ? (Le délai doit couvrir aussi les **écritures**.)
- **D6.** `KEYS *` sur un million de clés tient le verrou combien de temps ? Les autres clients
  attendent-ils pendant ce temps ? Proposer une solution (copie des clés sous verrou, filtrage hors
  verrou) et la mesurer.
- **D7.** Pendant l'arrêt gracieux, un client envoie une nouvelle commande sur une connexion encore
  ouverte. La traiter ou la refuser ?

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.**

```
projet-05/
├── go.mod
├── cmd/kvd/main.go           drapeaux, signal.NotifyContext, net.Listen, srv.Serve(ctx, ln)
└── internal/
    ├── store/
    │   ├── store.go          Store : map + RWMutex, Get/Set/Del/Incr/SetEx/Keys/Len
    │   └── janitor.go        goroutine de purge, arrêtée par contexte
    ├── protocol/
    │   ├── parse.go          Parse(line string) (Command, error)
    │   └── parse_fuzz_test.go
    └── server/
        ├── server.go         Serve : boucle d'acceptation, sémaphore de clients, arrêt
        └── conn.go           une connexion : bufio.Scanner (avec Buffer pour S9), délais, recover
```

L'idée directrice : chaque couche ignore celle du dessus. Le magasin ne sait pas qu'il y a un réseau ;
le protocole ne sait pas qu'il y a un magasin ; le serveur relie les deux. Les tests de `server` se
font avec un listener sur `127.0.0.1:0` (port choisi par le système) et de vrais clients `net.Dial` dans
le test.

---

## Tests à écrire

- `store` : table pour chaque commande ; `INCR` concurrent (100 × 1 000) sous `-race` ; expiration
  avec horloge fausse ; purge avec `synctest`.
- `protocol` : table de commandes valides et invalides ; fuzzing « ne panique jamais ».
- `server` : un client fait `SET`/`GET` ; 200 clients simultanés ; dépassement de la limite de clients
  (S8) ; délai d'inactivité (S7, avec un délai court en test) ; ligne trop longue (S9) ; arrêt gracieux
  pendant qu'une commande lente est en cours (S10) ; aucune goroutine restante après `Serve`.
- Un test de charge rapide : 50 clients × 10 000 commandes, mesure du débit (commandes/s).

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 12 spécifications sont-elles satisfaites, avec `nc` comme avec les tests ? |
| **C'est correct** | `-race` silencieux, aucune fuite de goroutine, arrêt gracieux réel, `INCR` atomique, les 7 difficultés traitées ou documentées. |
| **C'est idiomatique** | `context` propagé, `select` pour toutes les attentes, verrous encapsulés, erreurs typées, paquets découplés. |
| **Prêt pour la production** | Délais partout (lecture **et** écriture), limites (clients, taille de ligne), backpressure explicite, paniques contenues et journalisées, `STATS` exploitable. |

Rends ton code même incomplet : une conception discutée vaut mieux qu'un projet abandonné.
