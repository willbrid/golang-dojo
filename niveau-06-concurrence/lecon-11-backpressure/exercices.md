# Leçon 11 — Exercices

Code dans `mes-solutions/niveau-06/lecon-11/`. Les simulations utilisent `testing/synctest`.

## Exercices faciles

**E1 — Rejouer la simulation.**
Reproduire le tableau du cours, puis ajouter des lignes pour des files de 0 (channel non bufferisé),
1, 10 et 100. Tracer (même à la main) le nombre de refus et la latence maximale en fonction de la
taille de la file. Quelle forme ont les deux courbes ?

**E2 — La file non bornée qui dure.**
Avec une file de 1 000, porter la simulation à 2 000 tâches puis 5 000. Que devient la latence
maximale ? Extrapoler : au bout d'une heure de surcharge à ce rythme, combien de temps attendrait la
dernière tâche, et combien de tâches seraient en mémoire ?

**E3 — Ajouter un worker.**
Garder la file de 4 et passer à 3 workers (capacité 1 tâche/ms = débit d'arrivée), puis 4. Que
deviennent les refus et la latence ? Puis rendre les arrivées **irrégulières** (intervalle tiré au
hasard entre 0 et 2 ms, moyenne 1 ms, avec un générateur à graine fixe — N4 L3) en gardant 3
workers. Que se passe-t-il ? Pourquoi une capacité moyenne égale au débit moyen ne suffit-elle pas ?

**E4 — Attente bornée.**
Remplacer `Submit` par `SubmitWait` avec un `context.WithTimeout` de 5 ms pour chaque soumission
(l'arrivée suivante ne part qu'après). Comment évoluent le nombre de refus, la latence des acceptées,
et le **temps total** de la simulation ? Que se passe-t-il pour les arrivées qui suivent ?

**E5 — Le client parti.**
Ajouter à `task` un `ctx`. Si une tâche est prise par un worker alors que son `ctx` est déjà annulé
(le client a abandonné après 10 ms), le worker la saute. Simuler avec la file de 1 000 : combien de
tâches sont sautées ? Combien de travail inutile a-t-on évité ?

---

## Exercice intermédiaire — Une file à priorités avec délestage

Un service reçoit des requêtes `high` (paiements) et `low` (rapports).

```go
func (q *PQueue) Submit(ctx context.Context, p Priority, work func(context.Context)) error
```

**Contraintes :**
1. Deux channels bornés, un par priorité. Les workers prennent **toujours** une tâche `high` s'il y en
   a, sinon une `low`. *(Indice : un `select` avec `default` sur `high`, puis un `select` bloquant sur
   les deux — leçon 4, E1.)*
2. Quand la file `high` est pleine, une requête `high` peut **évincer** la plus ancienne requête `low`
   en attente, qui reçoit alors `ErrOverloaded`. *(Plus difficile ; une version sans éviction est
   acceptable si elle est documentée.)*
3. Les requêtes dont le `ctx` est annulé ne sont pas traitées.
4. Simulation `synctest` : 70 % `low`, 30 % `high`, débit d'arrivée 1,5 × la capacité. Vérifier que
   **toutes** les `high` sont servies avec une latence bornée, et que les refus portent sur les `low`.
5. Les `low` peuvent-elles être affamées pour toujours ? Proposer une parade (et la tester).

---

## Défi — Limiter la concurrence plutôt que la file

1. Réécrire le service avec `errgroup.SetLimit(n)` et `g.TryGo` au lieu d'une file et de workers :
   `TryGo` qui échoue = refus. Quelle différence de comportement par rapport à la file de 4 ? Que
   représente la « file » dans ce modèle ?
2. Mesurer les deux modèles avec la même simulation. Lequel donne la meilleure latence ? Le plus
   faible taux de refus ?
3. Quand préférer chacun ? Répondre en pensant à un serveur HTTP qui reçoit des requêtes de durées
   très variables (de 1 ms à 2 s).

---

## Quiz

1. Que se passe-t-il quand le débit d'arrivée dépasse durablement la capacité, si la file est non bornée ?
2. Quelles sont les trois issues possibles en surcharge ?
3. Pourquoi une file longue peut-elle rendre un service occupé à 100 % et utile à 0 % ?
4. Quelles sont les trois politiques d'admission, et leur code `select` ?
5. Comment dimensionner une file ?
6. Pourquoi le rejet doit-il être une erreur explicite ? En quoi devient-il en HTTP ?
7. Pourquoi vérifier `ctx.Err()` avant de commencer une tâche qui a attendu ?
8. Que mesurer en production pour voir une surcharge arriver ?
9. Un buffer augmente-t-il la capacité d'un service ?
10. Qu'est-ce qu'une tempête de réessais, et qui la provoque ?
