# Leçon 10 — Exercices

Code dans `mes-solutions/niveau-10/lecon-10/`, en partant du paquet `obs` de l'exemple.

## Exercices faciles

**E1 — La moyenne qui ment.**
Enregistrer 99 requêtes de 10 ms et 1 requête de 5 s. Que valent la moyenne et la médiane ? Que montre
l'histogramme ? Quelle valeur reflète l'expérience de l'utilisateur malchanceux ?

**E2 — Exposer `/metrics`.**
Ajouter une route `GET /metrics` qui écrit `reg.WriteText`. Doit-elle passer par le middleware (et donc
apparaître dans ses propres métriques) ? Doit-elle être accessible depuis Internet ? Proposer une
solution (un second serveur sur une autre adresse, N7 L9).

**E3 — L'identifiant hostile.**
Envoyer une requête avec `X-Request-ID` de 10 000 caractères, puis une avec des retours à la ligne
(`"abc\nfaux journal"`). Que devient la ligne de journal ? Corriger : longueur bornée **et** caractères
autorisés (lettres, chiffres, `-`).

**E4 — La cardinalité cachée.**
Un collègue ajoute l'étiquette `error` avec `err.Error()` comme valeur. Combien de séries après 1 000
erreurs « adhérent 42 introuvable », « adhérent 43 introuvable »… ? Proposer une étiquette correcte.

**E5 — Les dépendances.**
Écrire un décorateur de `http.RoundTripper` qui mesure RED pour **chaque appel sortant**, étiqueté par le
nom de la dépendance (`catalog`), la méthode, et le statut (ou `error` si l'appel a échoué). Pourquoi ne pas
étiqueter par l'URL appelée ?

---

## Exercice intermédiaire — Observer le service de prêts

Instrumenter le service des leçons 2 et 9 :

1. Le middleware de l'exemple sur toutes les routes ; `GET /metrics` sur une adresse d'administration.
2. Des métriques **métier** : `loans_total`, `loans_refused_total{reason=…}` (avec une liste **fermée** de
   raisons, dérivée des erreurs du domaine par `errors.Is`), `returns_late_days` (histogramme).
3. Les métriques du client résilient de la leçon 9 (appels, réessais, refus du disjoncteur et de la
   cloison) et de la base (USE à partir de `db.Stats()`, N8 L5).
4. Chaque ligne de journal porte l'identifiant de requête ; un test vérifie qu'une requête qui échoue
   produit une ligne avec l'identifiant, la route, le statut et la cause, **sans** le mot de passe ni le
   DSN.
5. Écrire en cinq lignes les trois alertes que tu mettrais en place, avec leur seuil et l'action attendue.

---

## Défi — Une trace minimale

Sans bibliothèque externe, propager une trace à travers deux services :

1. À l'entrée, lire ou créer un `traceparent` au format W3C (`00-<trace-id 32 hex>-<span-id 16 hex>-01`) ;
   chaque traitement crée un **span** (identifiant, parent, nom, début, durée).
2. Le `Transport` crée un span enfant pour chaque appel sortant et envoie un `traceparent` dont le parent est
   ce span.
3. Les spans terminés sont envoyés à un collecteur en mémoire. Afficher l'arbre d'une requête qui traverse
   « prêt » → « catalogue » → « base simulée », avec les durées.
4. Échantillonnage : ne conserver qu'une trace sur 10 (décision prise **à l'entrée**, propagée dans le
   dernier champ de `traceparent`). Pourquoi la décision doit-elle être prise une seule fois, à l'entrée ?
5. Comparer avec ce que fournit OpenTelemetry (niveau 12) : qu'est-ce qui reste à faire pour un usage réel ?

---

## Quiz

1. À quelle question répond chacun des trois signaux ?
2. Pourquoi instrumenter aux frontières, et lesquelles ?
3. Que mesurent RED et USE ?
4. Pourquoi un histogramme plutôt qu'une moyenne ?
5. Qu'est-ce qu'un identifiant de corrélation, et comment le propager ?
6. Qu'est-ce que la cardinalité, et quelles étiquettes faut-il éviter ?
7. Que contient `r.Pattern`, et quand est-il rempli ?
8. Pourquoi alerter sur les symptômes plutôt que sur les causes ?
9. Qu'est-ce qu'un objectif de service et un budget d'erreurs ?
10. Pourquoi l'observabilité est-elle une décision d'architecture ?
