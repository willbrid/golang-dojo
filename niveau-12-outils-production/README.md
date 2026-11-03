# Niveau 12 — Outils et production

**Objectif :** livrer, déployer, observer et déboguer un service Go en production : versionner et
construire le binaire, l'emballer dans une image, l'intégrer en continu, le configurer, le rendre
observable, le sécuriser, et l'exploiter.

**Prérequis :** niveaux 1 à 11. En particulier :
- les **modules** et le versionnement sémantique (N2 L9) ;
- l'**outillage** de qualité : `go vet`, `staticcheck`, `golangci-lint`, `govulncheck` (N5 L8) ;
- un **serveur HTTP de production** : délais, arrêt propre (N7 L7, N7 L9) ;
- l'**injection de dépendances** et `run(ctx, …)` (N10 L3) ;
- les **décisions d'observabilité** : quoi mesurer, cardinalité, corrélation (N10 L10) ;
- les **profils** et la mesure (niveau 11).

> **Ordre des leçons.** Du code source au service qui tourne. On versionne d'abord (1), puis on
> construit le binaire (2) et on l'emballe (3), et on automatise le tout (4). On rend ensuite le
> service configurable (5) et observable (6, 7). On le sécurise (8), et on l'exploite (9).

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Git et projets Go](lecon-01-git-projets/) | Organiser un dépôt, écrire des commits utiles, étiqueter des versions, lire la version d'un binaire, trouver une régression avec `git bisect` |
| 2 | [Build de production](lecon-02-build/) | Cross-compiler, injecter des valeurs avec `-ldflags`, produire des builds reproductibles |
| 3 | [Docker](lecon-03-docker/) | Écrire un Dockerfile multi-étapes, produire une image minimale et sûre |
| 4 | [CI/CD](lecon-04-ci-cd/) | Automatiser tests, lint, race, build et publication avec GitHub Actions |
| 5 | [Configuration](lecon-05-configuration/) | Lire la configuration de l'environnement, gérer la précédence, valider au démarrage |
| 6 | [Logs structurés](lecon-06-slog/) | Journaliser avec `log/slog` : niveaux, contexte, gestionnaires, ce qu'on ne journalise jamais |
| 7 | [Métriques et traces](lecon-07-metriques-traces/) | Exposer des métriques Prometheus, tracer avec OpenTelemetry, corréler logs, métriques et traces |
| 8 | [Sécurité et secrets](lecon-08-securite/) | Analyser les vulnérabilités, gérer les secrets, réduire la surface d'attaque |
| 9 | [Production](lecon-09-production/) | Déployer sans interruption, sondes de santé, déboguer un service en marche |

➡️ **[Projet 8 — Microservice Go professionnel](../projets/projet-08-microservice/)**

## Règle du niveau

**Ce qui n'est pas automatisé et vérifié finit par casser.** Un build, un déploiement ou une
configuration qui dépend d'une manipulation à la main est un incident qui attend son heure.
