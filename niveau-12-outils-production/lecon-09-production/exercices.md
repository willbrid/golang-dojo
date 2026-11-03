# Leçon 9 — Exercices

Code dans `mes-solutions/niveau-12/lecon-09/`, en partant du paquet `deploy` de l'exemple.

## Exercices faciles

**E1 — Prêt seulement quand c'est vrai.**
Ajouter au `Backend` un temps de démarrage (par exemple 500 ms pour « charger un cache ») pendant lequel
`/readyz` répond 503. Dans `rolling`, ajouter la nouvelle instance au répartiteur **immédiatement**, sans
attendre, puis arrêter l'ancienne dès que la nouvelle est déclarée prête. Combien d'erreurs ? Et si `/readyz`
répondait 200 dès le lancement, alors que le cache n'est pas chargé ?

**E2 — Combien attendre ?**
Avec une période de sonde de 100 ms, mesurer le nombre d'erreurs de l'arrêt propre pour une attente de
propagation de 0, 50, 100, 150, 300 ms (trois exécutions chacune). Où se situe le seuil ? Pourquoi faut-il une
marge au-dessus de la période ?

**E3 — Un arrêt borné.**
`Shutdown` peut attendre 5 s une connexion neuve inutilisée. Écrire un `Stop` qui garantit une durée
maximale : `Shutdown` avec un contexte limité, puis `Close` si le délai est dépassé. Le mesurer avec la
connexion neuve de `TestShutdownWithNewConn`. Qu'est-ce qui est perdu quand on passe à `Close` ?

**E4 — Les sondes et la base en panne.**
Écrire les deux sondes d'un service qui dépend d'une base (simulée par une interface `Pinger`). Quand la base
est en panne : `/livez` répond 200 ; `/readyz` répond 200 ; `GET /books/1` répond 503 avec un message clair.
Écrire la table de tests. Puis écrire la version « mauvaise » (la base vérifiée dans `/livez`) et décrire ce
que ferait un orchestrateur pendant une panne de la base de 30 s.

**E5 — Les goroutines sans `SIGQUIT`.**
Exposer `net/http/pprof` sur une adresse d'administration du programme `quit` de l'exemple, et lire
`/debug/pprof/goroutine?debug=2`. Écrire un petit outil qui compte les goroutines par état (`chan receive`,
`sleep`, `IO wait`…) et par fonction de départ. Le processus est-il toujours vivant après la lecture ?

---

## Exercice intermédiaire — Le cycle de vie complet

Écrire `run(ctx, args, env, stdout) error` pour le service de prêts (ou un service minimal), qui rassemble le
niveau :

1. charge et valide la configuration (leçon 5) et construit le logger (leçon 6), les métriques et les traces
   (leçon 7) ;
2. démarre le serveur d'administration (`/metrics`, `/debug/pprof/`, `/version`), puis le serveur public avec
   `/livez` et `/readyz` ;
3. `/readyz` ne passe à 200 qu'une fois la base joignable **au démarrage** ;
4. à l'annulation de `ctx` (`SIGTERM`) : `/readyz` → 503, attente de propagation (configurable),
   `Shutdown` avec délai, fermeture du pool de base et `TracerProvider.Shutdown`. Chaque étape est
   journalisée avec sa durée ;
5. un test lance `run` dans une goroutine, attend `/readyz` à 200, annule le contexte, et vérifie l'ordre des
   lignes de journal et que `run` renvoie `nil` dans le délai prévu.

---

## Défi — Le canari et le retour arrière automatique

1. Étendre le `Balancer` : chaque instance a un **poids**. La nouvelle version (le canari) reçoit 10 % du
   trafic.
2. Le canari a un défaut : il répond 500 à 5 % des requêtes. La version stable n'en a pas.
3. Écrire un contrôleur qui, toutes les 500 ms, compare le taux d'erreurs du canari à celui de la version
   stable (compteurs par instance, comme des métriques RED). Il **retire** le canari si l'écart est
   significatif. Attention : avec 10 % du trafic, combien de requêtes faut-il pour qu'une différence de 5 %
   ne soit pas du hasard ? Une règle simple, comme un nombre minimal de requêtes et un écart minimal, suffit ;
   la justifier.
4. Mesurer :
   - le temps entre le début du canari et sa mise à l'écart ;
   - le nombre d'erreurs vues par les clients ;
   - le nombre de fausses alertes sur 20 essais avec un canari **sans** défaut.
5. Comparer avec un déploiement progressif sans canari de la même version défectueuse : combien d'erreurs ?

---

## Quiz

1. Qu'est-ce qu'un déploiement progressif, et quelle contrainte impose-t-il au schéma de la base ?
2. Quelle différence entre sonde de vie et sonde de disponibilité ? Que se passe-t-il quand chacune échoue ?
3. Pourquoi ne jamais vérifier une dépendance dans la sonde de vie ?
4. Dans quel ordre une instance doit-elle s'arrêter ? Pourquoi `Shutdown` seul ne suffit-il pas ?
5. Comment les délais d'arrêt doivent-ils s'emboîter ?
6. Pourquoi un arrêt peut-il durer 5 secondes de plus que prévu ?
7. Que faut-il pour qu'un retour arrière soit possible en quelques minutes ?
8. Qu'est-ce qu'un déploiement canari ?
9. Comment voir les goroutines d'un service en production sans l'arrêter ?
10. Pourquoi ne jamais corriger une instance à la main pendant un incident ?
