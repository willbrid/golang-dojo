# Leçon 5 — Pool de connexions

> Un service en panne de base de données ne plante pas : il **ralentit**, puis cesse de répondre, sans
> une seule erreur dans les journaux de la base. Les requêtes attendent une connexion libre qui ne
> vient pas, parce que le pool est trop petit, ou parce qu'un `Rows` oublié en retient une à chaque
> appel. À l'inverse, un pool trop grand fait tomber la base elle-même. Cette leçon apprend à régler le
> pool, à lire ses statistiques et à trouver une fuite.

## Objectifs

1. Connaître les quatre réglages du pool de `database/sql` et leurs valeurs par défaut.
2. Dimensionner le pool en fonction de la base, du nombre d'instances et de la charge.
3. Reconnaître une **saturation** et une **fuite** dans `db.Stats()` et `pg_stat_activity`.
4. Comprendre pourquoi l'annulation du contexte libère une connexion retenue.
5. Écrire un test qui échoue quand une fonction fuit une connexion.

## Explication

### Le cycle d'une connexion

Quand une requête a besoin d'une connexion, le pool :

1. prend une connexion **inactive** s'il y en a une ;
2. sinon, en **ouvre** une nouvelle, si le plafond `MaxOpenConns` n'est pas atteint ;
3. sinon, **attend** qu'une connexion soit rendue, ou que le contexte de la requête se termine.

Quand la requête rend la connexion (fin du `Scan`, `rows.Close()`, `Commit`…), le pool la garde
inactive, ou la **ferme** s'il en a déjà assez en réserve (`MaxIdleConns`) ou si elle est trop vieille.

### Les quatre réglages

| Réglage | Défaut | Rôle |
|---|---|---|
| `SetMaxOpenConns(n)` | **illimité** | Plafond de connexions ouvertes (utilisées + inactives). |
| `SetMaxIdleConns(n)` | **2** | Connexions inactives gardées en réserve ; les autres sont fermées dès qu'elles sont rendues. |
| `SetConnMaxLifetime(d)` | illimité | Âge maximal d'une connexion ; au-delà, elle est fermée à son prochain retour au pool. |
| `SetConnMaxIdleTime(d)` | illimité | Durée maximale d'inactivité avant fermeture. |

Les deux valeurs par défaut sont mauvaises pour un service :

- **Illimité** : sous une pointe de 300 requêtes simultanées, le pool ouvre 300 connexions. PostgreSQL en
  accepte 100 par défaut (`max_connections`, dont 3 réservées aux superutilisateurs) et refuse les
  suivantes avec `FATAL: sorry, too many clients already (SQLSTATE 53300)` — vu à la leçon 4, où
  173 virements sur 300 avaient échoué ainsi.
- **2 inactives** : après chaque pointe, toutes les connexions sauf deux sont fermées, puis rouvertes à la
  pointe suivante. Mesuré, 50 rafales de 20 requêtes `SELECT 1` avec 20 connexions au plus :

  | `MaxIdleConns` | Durée | Connexions fermées puis rouvertes |
  |---|---|---|
  | 2 (défaut) | 1,23 s | ≈ 870 |
  | 20 | 45 ms | 0 |

  Ouvrir une connexion PostgreSQL coûte une poignée de main TCP, éventuellement TLS, et une
  authentification (≈ 4,5 ms mesurés en local, bien plus à travers un réseau). Règle simple :
  **`MaxIdleConns` = `MaxOpenConns`**, et laisser `ConnMaxIdleTime` fermer les connexions vraiment inutiles.

`ConnMaxLifetime` (de l'ordre de 30 minutes à une heure) renouvelle les connexions : cela répartit la
charge après l'ajout d'un réplica, suit un changement d'adresse DNS ou une bascule de base, et libère la
mémoire que chaque processus serveur PostgreSQL accumule.

### Dimensionner

Trois contraintes :

1. **La base** : `instances × MaxOpenConns ≤ max_connections − marge` (connexions d'administration,
   migrations, outils). Dix instances d'un service à 20 connexions = 200 : trop pour une base à 100.
2. **Le besoin** : le nombre moyen de connexions occupées vaut à peu près
   *débit × durée moyenne d'une requête*. 500 requêtes SQL/s de 4 ms occupent en moyenne 2 connexions ;
   on prévoit de la marge pour les pointes et la variance (le niveau 11 donne le nom de cette relation,
   la loi de Little).
3. **Le serveur de base** : au-delà de quelques connexions **actives** par cœur, PostgreSQL passe son
   temps à commuter entre requêtes. Plus de connexions ne fait pas plus de travail ; cela fait plus
   d'attente, **dans** la base au lieu de **dans** le pool.

Un pool plus petit que la demande n'est pas un défaut : c'est de la **backpressure** (N6 L11). Les
requêtes attendent dans le pool, chacune avec son délai (`context`), au lieu de surcharger la base.

Point de départ raisonnable pour un service web : `MaxOpenConns` entre 10 et 25 par instance, ajusté
en lisant les statistiques. Avec pgx natif (`pgxpool`), les réglages s'appellent `MaxConns` (défaut :
le plus grand de 4 et du nombre de processeurs), `MinConns`, `MaxConnLifetime` (1 h) et
`MaxConnIdleTime` (30 min), et se passent dans le DSN (`pool_max_conns=20`) ou par `pgxpool.Config`.

### Lire `db.Stats()`

| Champ | Lecture |
|---|---|
| `OpenConnections`, `InUse`, `Idle` | l'état instantané |
| `WaitCount`, `WaitDuration` | combien de requêtes ont **attendu** une connexion, et combien de temps au total : la saturation |
| `MaxIdleClosed` | fermées faute de place en réserve : `MaxIdleConns` trop petit |
| `MaxIdleTimeClosed`, `MaxLifetimeClosed` | fermées par les délais : normal |

Les compteurs sont **cumulatifs** : on les exporte comme métriques (niveau 12) et on regarde leur
**variation**. Diagnostic :

- `WaitDuration` qui grimpe, `InUse = MaxOpenConns`, et une base **calme** (`pg_stat_activity` : peu de
  connexions `active`) → les connexions sont **retenues sans travailler** : une fuite ;
- `WaitDuration` qui grimpe, base **saturée** (requêtes lentes, CPU élevé) → le problème est dans la
  base (index manquant, leçon 8), pas dans le pool. L'agrandir aggraverait les choses ;
- `InUse` qui augmente et ne redescend jamais, même la nuit → une fuite.

### Ce qui retient une connexion

- un `*sql.Rows` non fermé (lu partiellement, `return` anticipé) ;
- un `*sql.Row` dont on n'a pas appelé `Scan` ;
- une transaction sans `Commit` ni `Rollback` ;
- un `*sql.Conn` non fermé (leçon 1).

**L'annulation du contexte libère la connexion** : `database/sql` surveille le contexte d'une requête et
ferme les `Rows` (ou annule la transaction) quand il se termine. C'est un filet de sécurité important :
dans un handler HTTP, le contexte de la requête se termine quand le handler rend la main (N7 L7), donc un
`Rows` oublié n'est retenu « que » le temps de la requête. Mais dans une **tâche de fond** qui utilise
`context.Background()`, rien ne vient le libérer : la connexion est perdue pour toujours. Ce filet ne
remplace pas `defer rows.Close()` : pendant que la requête HTTP dure, la connexion est retenue, et
sous charge, cela suffit à saturer le pool.

## Exemple

La même fonction boguée, appelée avec un contexte éternel puis avec un contexte annulable, et la
version correcte.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"os"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// firstTitleLeaky a un bug : sur le chemin « trouvé », rows n'est jamais fermé.
func firstTitleLeaky(ctx context.Context, db *sql.DB) (string, error) {
	rows, err := db.QueryContext(ctx, `SELECT title FROM l5_books ORDER BY id`)
	if err != nil {
		return "", err
	}
	if rows.Next() {
		var t string
		err := rows.Scan(&t)
		return t, err // ← retour anticipé : rows reste ouvert, et sa connexion avec
	}
	rows.Close()
	return "", rows.Err()
}

func firstTitle(ctx context.Context, db *sql.DB) (string, error) {
	rows, err := db.QueryContext(ctx, `SELECT title FROM l5_books ORDER BY id`)
	if err != nil {
		return "", err
	}
	defer rows.Close() // tous les chemins
	if rows.Next() {
		var t string
		err := rows.Scan(&t)
		return t, err
	}
	return "", rows.Err()
}

func newPool() *sql.DB {
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	db.SetMaxOpenConns(5)                   // plafond : jamais plus de 5 connexions
	db.SetMaxIdleConns(5)                   // garder les 5 ouvertes entre deux pics
	db.SetConnMaxLifetime(30 * time.Minute) // renouveler régulièrement
	db.SetConnMaxIdleTime(5 * time.Minute)  // fermer celles qui ne servent plus
	return db
}

func report(label string, db *sql.DB) {
	s := db.Stats()
	fmt.Printf("%-40s ouvertes=%d utilisées=%d attentes=%d\n", label, s.OpenConnections, s.InUse, s.WaitCount)
}

// waitIdle attend (au plus 1 s) que plus aucune connexion ne soit utilisée.
func waitIdle(db *sql.DB) {
	for deadline := time.Now().Add(time.Second); db.Stats().InUse > 0 && time.Now().Before(deadline); {
		time.Sleep(10 * time.Millisecond)
	}
}

func main() {
	bg := context.Background()
	setup := newPool()
	if _, err := setup.ExecContext(bg, `
		DROP TABLE IF EXISTS l5_books;
		CREATE TABLE l5_books (id int PRIMARY KEY, title text NOT NULL);
		INSERT INTO l5_books VALUES (1, 'L''Étranger'), (2, 'La Peste')`); err != nil {
		log.Fatal(err)
	}
	setup.Close()

	// 1. Une tâche de fond, avec un contexte qui ne se termine jamais.
	leaky := newPool()
	for range 5 {
		firstTitleLeaky(bg, leaky)
	}
	report("fuite : après 5 appels", leaky)
	ctx, cancel := context.WithTimeout(bg, 200*time.Millisecond)
	_, err := firstTitle(ctx, leaky) // même la version correcte ne trouve plus de connexion
	cancel()
	fmt.Println("fuite : 6e appel →", err)
	report("fuite : après le 6e appel", leaky)

	// 2. Même bug, mais avec un contexte annulable (celui d'une requête HTTP, par exemple).
	pool := newPool()
	defer pool.Close()
	reqCtx, endRequest := context.WithCancel(bg)
	for range 5 {
		firstTitleLeaky(reqCtx, pool)
	}
	report("contexte annulable : après 5 appels", pool)
	endRequest() // database/sql ferme les Rows dont le contexte est terminé
	waitIdle(pool)
	report("contexte annulable : après annulation", pool)

	// 3. La version correcte, 20 appels.
	for range 20 {
		firstTitle(bg, pool)
	}
	report("correct : après 20 appels", pool)
}
```

Sortie :

```
fuite : après 5 appels                   ouvertes=5 utilisées=5 attentes=0
fuite : 6e appel → context deadline exceeded
fuite : après le 6e appel                ouvertes=5 utilisées=5 attentes=1
contexte annulable : après 5 appels      ouvertes=5 utilisées=5 attentes=0
contexte annulable : après annulation    ouvertes=5 utilisées=0 attentes=0
correct : après 20 appels                ouvertes=5 utilisées=0 attentes=0
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `firstTitleLeaky` | Le `return` à l'intérieur du `if rows.Next()` saute le `rows.Close()` placé plus bas. |
| `newPool` | Les quatre réglages explicites : plafond, réserve égale au plafond, renouvellement, inactivité. |
| `utilisées=5` après 5 appels | Chaque appel a laissé sa connexion retenue par un `Rows` ouvert. |
| 6ᵉ appel → `context deadline exceeded` | Même la version **correcte** échoue : elle attend une connexion qui ne viendra pas. Le symptôme apparaît **ailleurs** que le bug. |
| `attentes=1` | `WaitCount` : la saturation devient visible dans les statistiques. |
| `endRequest()` puis `utilisées=0` | À l'annulation du contexte, `database/sql` ferme les `Rows` concernés (de façon asynchrone, d'où `waitIdle`). |
| `firstTitle` + `defer rows.Close()` | 20 appels, aucune connexion retenue. |
| `leaky` jamais fermé | Fermer le pool ne répare rien : `db.Close()` rend la main tout de suite (mesuré, même avec un `Rows` ouvert) et empêche les nouvelles requêtes ; les connexions retenues ne sont fermées que lorsqu'on les rend, c'est-à-dire jamais ici. |

## Erreurs fréquentes

1. **Garder les valeurs par défaut** : pool illimité (base saturée en pointe) et 2 connexions en réserve
   (réouvertures permanentes).
2. **`MaxIdleConns` > `MaxOpenConns`** : sans effet (`database/sql` le ramène au plafond), mais signe d'un
   réglage non compris.
3. **Agrandir le pool pour corriger une fuite** : la fuite épuise le nouveau pool, un peu plus tard.
4. **Agrandir le pool quand la base est saturée** : plus de concurrence dans une base déjà débordée.
5. **Oublier le nombre d'instances** dans le calcul.
6. **Des requêtes sans délai** : en cas de saturation, elles attendent une connexion indéfiniment, et les
   goroutines s'accumulent.
7. **`context.Background()` dans le code de requête** : le filet de l'annulation disparaît.
8. **Ne pas exporter `db.Stats()`** : la saturation n'est visible qu'au moment de la panne.

## Bonnes pratiques Go

- Régler les quatre valeurs explicitement, depuis la configuration (niveau 12).
- `MaxIdleConns = MaxOpenConns` ; `ConnMaxLifetime` ≈ 30 min – 1 h ; `ConnMaxIdleTime` ≈ quelques minutes.
- `instances × MaxOpenConns` nettement sous `max_connections`.
- Chaque requête porte le contexte de son appelant, avec un délai.
- Exporter `db.Stats()` en métriques ; alerter sur `WaitDuration`.
- Dans les tests, vérifier `db.Stats().InUse == 0` à la fin (exercice intermédiaire).

## Ce que je dois retenir

- `*sql.DB` réutilise, ouvre, ou **fait attendre** ; les défauts (illimité, 2 en réserve) ne conviennent
  pas à un service.
- Dimensionner avec la base, les instances et *débit × durée* ; un pool borné est de la backpressure.
- `WaitCount`/`WaitDuration` = saturation ; `InUse` qui ne redescend pas = fuite.
- `Rows` non fermé, `Row` non scanné, transaction non terminée, `Conn` non fermé retiennent une connexion.
- L'annulation du contexte libère la connexion ; `context.Background()` supprime ce filet.

---

➡️ [Exercices](exercices.md)
