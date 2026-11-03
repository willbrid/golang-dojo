# Niveau 10 — Architecture

**Objectif :** structurer un service Go professionnel — séparer les responsabilités, assembler les
dépendances, isoler le domaine, découper en modules, communiquer par événements, résister aux pannes,
se rendre observable — et, à chaque fois, savoir **quand ne pas le faire**.

**Prérequis :** niveaux 1 à 9. En particulier :
- les **paquets** et `internal/` (N2 L8, N2 L10) : l'architecture d'un programme Go se lit dans ses imports ;
- les **interfaces côté consommateur** (N2 L4, N5 L3) et les faux de test ;
- les **erreurs HTTP** (N7 L5), l'**arrêt gracieux** (N7 L7) et les **transactions** (N8 L4) ;
- le *repository* et la suite de contrat (N8 L7) ;
- la **concurrence** et la **backpressure** (niveau 6) pour les leçons 8 et 9.

> **Fil rouge.** Toutes les leçons utilisent le même domaine : une bibliothèque de prêt (adhérents,
> emprunts, retours, retards, notifications). Il est assez petit pour tenir dans un exemple, et assez riche
> pour avoir de vraies règles, des événements et des appels à des services externes.

> **Ordre des leçons.** Du plus local au plus global : séparer (1), le découpage par défaut d'un backend
> (2), assembler (3), concevoir les interfaces qui relient les parties (4). Puis les architectures
> nommées (5, 6) et le découpage d'un système (7). Enfin ce qui traverse les frontières : messages (8),
> pannes (9), observabilité (10).

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Séparation des responsabilités](lecon-01-separation/) | Isoler les règles métier du transport et du stockage ; lire les dépendances |
| 2 | [handler / service / repository](lecon-02-handler-service-repository/) | Placer validation, autorisation, transactions et traduction d'erreurs au bon endroit |
| 3 | [Injection de dépendances](lecon-03-injection-dependances/) | Assembler un programme dans `main`, par constructeurs, sans framework |
| 4 | [Interfaces bien conçues](lecon-04-interfaces/) | Écrire de petites interfaces côté consommateur, les découvrir au lieu de les prévoir |
| 5 | [Hexagonale et clean](lecon-05-hexagonale-clean/) | Ports et adaptateurs ; mesurer leur coût réel |
| 6 | [Domain-Driven Design](lecon-06-ddd/) | Valeurs, entités, agrégats et invariants ; contextes bornés |
| 7 | [Monolithe modulaire et microservices](lecon-07-monolithe-modulaire/) | Découper en modules aux frontières vérifiées ; savoir quand extraire un service |
| 8 | [Événementiel et messaging](lecon-08-evenementiel/) | Outbox transactionnelle, livraison au moins une fois, consommateurs idempotents |
| 9 | [Résilience](lecon-09-resilience/) | Idempotence, réessais avec *jitter*, disjoncteur, budgets de délai |
| 10 | [Observabilité](lecon-10-observabilite/) | Décider quoi journaliser, mesurer et tracer, et où |

## Évaluation de fin de niveau

Le **[projet 7 — Workers et queues](../projets/projet-07-workers-queues/)**.

## Règle du niveau

**Chaque abstraction doit payer sa place.** Pour chaque couche, interface ou module ajouté, pouvoir dire
quel test, quel changement ou quelle panne il rend plus facile. Sinon, c'est de la cérémonie.
