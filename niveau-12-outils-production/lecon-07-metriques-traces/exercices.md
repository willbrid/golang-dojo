# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-12/lecon-07/`, en partant du paquet `obs` et de la démonstration du cours.

## Exercices faciles

**E1 — Une métrique métier et une jauge calculée.**
Ajouter au catalogue un compteur `books_viewed_total` (sans étiquette d'identifiant !) et une jauge
`cache_entries` dont la valeur est **lue** au moment de l'interrogation (`prometheus.NewGaugeFunc`). Pourquoi
une `GaugeFunc` plutôt qu'un `Set` à chaque modification du cache ? Vérifier les deux dans un test avec
`testutil`.

**E2 — Les bornes et le percentile.**
Écrire une fonction qui calcule un percentile à partir des compteurs d'un histogramme, comme
`histogram_quantile` (interpolation linéaire dans l'intervalle qui contient le rang). Simuler 100 000
durées tirées entre 80 et 160 ms. Calculer le vrai p99 (en triant), puis le p99 estimé avec les bornes par
défaut de Prometheus (`prometheus.DefBuckets`), puis avec les bornes du cours. Quelle erreur dans chaque cas ?

**E3 — L'explosion de cardinalité.**
Ajouter une étiquette `user_id` au compteur de requêtes et simuler 10 000 utilisateurs. Combien de séries
(`testutil.CollectAndCount`) ? Quelle taille fait la réponse de `/metrics` ? Combien de temps prend-elle à
produire ? Comparer avec la version sans `user_id`.

**E4 — La trace cassée.**
Dans la démonstration, créer la requête sortante avec `http.NewRequest` au lieu de
`http.NewRequestWithContext(r.Context(), …)`. Que deviennent les spans du stock (trace, parent) ? Et les
journaux ? Pourquoi aucune erreur ne le signale-t-elle ?

**E5 — L'échantillonnage incohérent.**
Configurer le catalogue à 50 % et le stock à 10 %, d'abord avec `TraceIDRatioBased` seul, puis avec
`ParentBased(TraceIDRatioBased(…))`. Envoyer 1 000 requêtes et compter :
- les traces gardées dans chaque service ;
- les traces complètes (présentes dans les deux) ;
- les spans du stock dont le parent est absent.

Qu'est-ce qui rend les traces cohérentes, la même décision prise dans les deux services ?

---

## Exercice intermédiaire — Le service de prêts, observable

Pour le service de prêts du niveau 10 (ou ton projet 4/7) :

1. Remplacer le registre écrit à la main de N10 L10 par Prometheus : RED par middleware, plus deux
   métriques métier (prêts créés, prêts refusés par motif, avec un **petit** ensemble de motifs).
2. Ajouter `otelhttp` en entrée et sur tout client HTTP sortant, et un span autour de chaque méthode du dépôt
   (N8 L7), avec l'attribut `db.operation` (`INSERT`, `SELECT`…). Jamais la requête SQL complète avec ses
   valeurs.
3. Brancher `TraceHandler` sur le logger de la leçon 6. Vérifier qu'une ligne d'erreur du dépôt porte le même
   `trace_id` que la ligne de requête du middleware.
4. Un test vérifie, avec un exportateur en mémoire (`tracetest.NewInMemoryExporter`) :
   - une requête `POST /loans` produit un span serveur, un span de dépôt, et un lien parent-enfant entre eux ;
   - aucun attribut ne contient de donnée personnelle.
5. Mesurer le coût de l'instrumentation de bout en bout sur cette route, avec le générateur de N11 L7.

---

## Défi — Prometheus et Jaeger pour de vrai

1. Écrire un `compose.yaml` qui lance :
   - le catalogue et le stock, chacun dans son conteneur (leçon 3) ;
   - Prometheus (`prom/prometheus`), qui interroge leurs `/metrics` toutes les 5 s ;
   - Jaeger (`jaegertracing/jaeger`, qui reçoit OTLP).
2. Remplacer l'exportateur stdout par un exportateur OTLP (`otlptracehttp` ou `otlptracegrpc`), dont
   l'adresse vient de la configuration (`OTEL_EXPORTER_OTLP_ENDPOINT`).
3. Envoyer de la charge. Dans Prometheus, écrire et exécuter les requêtes :
   - le débit par route : `sum by (route) (rate(http_requests_total[1m]))` ;
   - le taux d'erreurs ;
   - le p99 par route avec `histogram_quantile`.
4. Rendre le stock lent pour un livre sur dix. Retrouver une trace lente dans Jaeger, puis les journaux de
   cette trace.
5. Écrire une règle d'alerte Prometheus sur le p99, et vérifier qu'elle se déclenche (`/api/v1/alerts`).

---

## Quiz

1. Comment Prometheus obtient-il les métriques d'un service ?
2. Compteur, jauge, histogramme : à quoi sert chacun ?
3. Pourquoi exposer un compteur plutôt qu'un taux ?
4. Pourquoi les bornes d'un histogramme comptent-elles ?
5. Pourquoi préférer un histogramme à un *summary* pour un service à plusieurs instances ?
6. Pourquoi un registre à soi plutôt que le registre global ?
7. Qu'est-ce qu'un span ? Qu'est-ce qui relie les spans de deux services ?
8. Pourquoi `ParentBased` ?
9. Pourquoi appeler `TracerProvider.Shutdown` ?
10. Comment relier une alerte, une trace et des journaux ?
