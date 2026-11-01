# Niveau 7 — Réseau, HTTP et services

**Objectif :** écrire des services HTTP corrects, sûrs et testables avec la bibliothèque standard —
serveur, client, middleware, conception d'API, erreurs, authentification, mise en production — puis
descendre aux couches inférieures (TCP, TLS) et monter vers gRPC.

**Prérequis :** niveaux 1 à 6. En particulier :
- les **interfaces** (N2 L4) : `http.Handler` est une interface d'une méthode ;
- **Readers/Writers** (N4 L5) et **JSON** (N4 L6) : le corps d'une requête est un `io.Reader`,
  la réponse un `io.Writer` ;
- les **tests** (niveau 5) : chaque leçon s'appuie sur `net/http/httptest` ;
- la **concurrence** (niveau 6) : chaque requête est servie dans sa propre goroutine, et `context`
  traverse tout le niveau.

> **Ordre des leçons.** Le serveur d'abord (leçons 1 à 5), parce qu'on y écrit le plus de code ; le
> client ensuite (6), puis le contexte de bout en bout (7) qui relie les deux. L'authentification (8)
> et la mise en production (9) supposent un serveur complet. TCP et TLS (10, 11) descendent sous HTTP ;
> gRPC (12) remonte au-dessus. La leçon 10 approfondit ce que le projet 5 utilisait déjà de `net`.

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [`net/http` — le serveur](lecon-01-serveur/) | Écrire des handlers, router avec `ServeMux` (motifs 1.22+), tester avec `httptest` |
| 2 | [Requêtes et réponses](lecon-02-requetes-reponses/) | Lire paramètres, en-têtes et corps ; écrire statuts, en-têtes et corps ; streamer |
| 3 | [Middleware](lecon-03-middleware/) | Composer journalisation, récupération de panique, identifiant de requête |
| 4 | [Conception d'API REST](lecon-04-api-rest/) | Ressources, verbes, statuts, pagination, versionnement |
| 5 | [Erreurs HTTP](lecon-05-erreurs-http/) | Un modèle d'erreur unique, du domaine au statut, sans rien fuiter |
| 6 | [Client HTTP](lecon-06-client/) | `http.Client`, `Transport`, délais, réessais, drainage des corps |
| 7 | [`context` de bout en bout](lecon-07-context/) | Propager l'annulation client → serveur → base ; arrêt gracieux |
| 8 | [Authentification](lecon-08-authentification/) | Mots de passe, sessions, jetons, clés d'API |
| 9 | [Serveur en production](lecon-09-production/) | Délais, limites de taille, reverse proxy, en-têtes de sécurité |
| 10 | [TCP et UDP](lecon-10-tcp-udp/) | Protocoles binaires, *framing*, DNS |
| 11 | [TLS et WebSockets](lecon-11-tls-websockets/) | Certificats, mTLS, mise à niveau de connexion |
| 12 | [RPC et gRPC](lecon-12-grpc/) | Protobuf, streaming, intercepteurs, quand préférer gRPC |

## Évaluation de fin de niveau

Le **[projet 3 — API REST avec tests](../projets/projet-03-api-rest/)**.

## Règle du niveau

Chaque handler a son test `httptest`. Un handler non testé n'est pas terminé.
