# Leçon 11 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé. Chiffres mesurés avec `synctest` (temps virtuel).

## E1 — Rejouer la simulation

2 workers, tâches de 3 ms, une arrivée par milliseconde, 200 arrivées :

| File | Refusées | Latence médiane | Latence max |
|---|---|---|---|
| 0 (non bufferisée) | 88 | 3 ms | 3 ms |
| 1 | 66 | 4 ms | 5 ms |
| 4 | 63 | 8 ms | 9 ms |
| 10 | 57 | 17 ms | 18 ms |
| 50 | 16–17 | 48–49 ms | 78 ms |
| 100 | 0 | 53 ms | 102 ms |
| 1 000 | 0 | 53 ms | 102 ms |

Les deux courbes vont en sens opposé : les **refus** diminuent quand la file grandit, la **latence
maximale** augmente à peu près **linéairement** avec la taille de la file (≈ taille / 2 workers × 3 ms).
Au-delà de 100, la file n'est jamais pleine sur 200 arrivées : plus de refus, mais une latence qui ne
dépend plus que de la durée de la surcharge. Il n'existe pas de taille qui donne « peu de refus **et**
faible latence » : en surcharge, il faut choisir.

## E2 — La file non bornée qui dure

La latence maximale croît **sans limite** avec le nombre de tâches : la file grossit de
λ − μ = 1 − 0,67 ≈ 0,33 tâche par milliseconde. Sur une heure (3 600 000 ms), environ 1,2 million de
tâches s'accumulent, et la dernière attend 1,2 million × 3 ms / 2 ≈ **30 minutes**. Chaque tâche en
attente garde en mémoire sa requête, ses buffers, son contexte… Le service tombera par manque de
mémoire bien avant — et en attendant, il traite des requêtes dont les clients sont partis depuis
longtemps.

## E3 — Ajouter un worker

Mesuré avec la file de 4 et des arrivées **régulières** :

| Workers | Refusées | Médiane | Max |
|---|---|---|---|
| 3 | 0 | 3 ms | 3 ms |
| 4 | 0 | 3 ms | 3 ms |

Avec des arrivées parfaitement régulières, 3 workers suffisent : chaque tâche trouve un worker libre
à son arrivée. Avec des arrivées **irrégulières** (intervalle uniforme dans [0, 2 ms], même moyenne),
3 workers, file de 4 :

| Tâches | Refusées | Médiane | Max |
|---|---|---|---|
| 200 | 4 | 3,9 ms | 7,2 ms |
| 2 000 | 70 | 4,5 ms | 7,6 ms |

Capacité moyenne = débit moyen ne suffit pas : les **rafales** remplissent la file plus vite qu'elle
ne se vide, et la file courte déborde. Plus on approche des 100 % d'occupation, plus l'attente et les
refus augmentent — c'est le résultat central de la théorie des files d'attente (niveau 11, leçon 7).
En pratique, on dimensionne pour une occupation de 60 à 80 %, pas 100 %.

## E4 — Attente bornée

Avec `SubmitWait` et 5 ms d'attente : moins de refus qu'avec `Submit` (une place se libère souvent
dans les 5 ms), latence des acceptées un peu plus élevée, et surtout **le producteur ralentit** — il
reste bloqué jusqu'à 5 ms par soumission, donc les arrivées suivantes sont **retardées**. C'est la
contre-pression au sens propre : la surcharge remonte vers l'amont sous forme de lenteur, au lieu de
s'accumuler dans la file. Si l'amont est un client HTTP, c'est sa latence qui augmente, ce qui est
souvent préférable à une erreur — jusqu'à son propre délai.

## E5 — Le client parti

```go
for t := range q.tasks {
	if t.ctx.Err() != nil {      // le client a abandonné pendant l'attente
		t.done <- 0               // ou une erreur explicite
		continue
	}
	t.work()
	…
}
```

Avec la file de 1 000 et un abandon client à 10 ms, toutes les tâches qui attendent plus de 10 ms
sont sautées : dans la simulation du cours, cela concerne la grande majorité des tâches (médiane
53 ms). Le service retrouve presque immédiatement sa capacité pour les clients **encore présents**.
C'est la mesure la plus rentable contre l'effondrement : ne jamais faire un travail dont personne
n'attend le résultat.

## Exercice intermédiaire — File à priorités

Le cœur du worker :

```go
for {
	select {                        // 1. priorité : une tâche high si elle est là
	case t := <-q.high:
		q.run(t)
		continue
	default:
	}
	select {                        // 2. sinon, la première disponible
	case t := <-q.high:
		q.run(t)
	case t := <-q.low:
		q.run(t)
	case <-q.quit:
		return
	}
}
```

- Le premier `select` avec `default` donne la priorité ; le second évite l'attente active (pas de
  `default`) tout en restant réveillable par une `high`.
- **Éviction** : `Submit(high)` sur une file `high` pleine tente `select { case t := <-q.low: t.reject(ErrOverloaded) … }`
  puis réessaie : la place libérée est dans `low`, pas dans `high` — l'éviction utile consiste plutôt à
  laisser la `high` **attendre** brièvement (`SubmitWait`) pendant qu'on rejette des `low`. La version
  sans éviction, documentée, est le choix raisonnable.
- **Famine des `low`** : si les `high` arrivent sans interruption, les `low` ne sont jamais servies.
  Parades : réserver un worker aux `low` ; ou servir une `low` toutes les N tâches ; ou rejeter les
  `low` trop vieilles plutôt que de les laisser vieillir indéfiniment. La simulation doit vérifier
  qu'une `low` admise finit par être servie (ou rejetée explicitement) dans un délai borné.

## Défi — Limiter la concurrence plutôt que la file

1. Avec `SetLimit(2)` et `TryGo`, il n'y a **plus de file** : une tâche est lancée immédiatement si
   moins de 2 tournent, refusée sinon. C'est le cas « file de 0 » du tableau d'E1 : 88 refus, latence
   minimale (3 ms). La « file » est réduite aux tâches en cours.
2. Concurrence limitée : meilleure latence, plus de refus. File courte : quelques refus de moins,
   latence un peu plus haute. Les deux bornent la latence ; la file absorbe en plus de petites rafales.
3. Pour un serveur HTTP aux durées très variables (1 ms à 2 s), limiter la **concurrence** est souvent
   plus robuste : une file de taille fixe peut contenir 4 requêtes de 2 s (8 s d'attente) ou 4 de 1 ms
   — la taille ne dit rien de la latence. En pratique : limite de concurrence + petite file + rejet des
   requêtes qui ont attendu trop longtemps.

## Quiz

1. Elle grossit sans limite ; la latence et la mémoire croissent jusqu'à l'effondrement.
2. Ralentir l'amont, refuser une partie du travail, augmenter la capacité.
3. Il passe son temps à traiter des requêtes dont les clients ont abandonné.
4. Bloquer (`q <- t`), attendre un temps borné (`select` avec `ctx.Done()`), rejeter (`select` avec `default`).
5. À partir de la latence maximale acceptable : taille ≈ latence × workers / temps de traitement.
6. Pour que l'appelant la reconnaisse (`errors.Is`) et réagisse ; en HTTP : 503 ou 429 avec `Retry-After`.
7. Pour ne pas faire un travail dont le client n'attend plus le résultat.
8. Longueur des files, nombre de refus, percentiles élevés de latence (p99), taux d'occupation.
9. Non : il absorbe des rafales, il ne traite pas plus vite.
10. Des clients qui réessaient immédiatement et en masse après des refus, aggravant la surcharge ;
    provoquée par des réessais sans attente aléatoire croissante.
