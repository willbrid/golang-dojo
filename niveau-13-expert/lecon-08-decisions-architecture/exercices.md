# Module 8 — Exercices

Code dans `mes-solutions/niveau-13/module-08/`, ADR dans `mes-solutions/niveau-13/module-08/adr/`. Les
exercices de ce module produisent surtout des **textes** : ADR, revues, réponses. Ils se jugent comme le reste
du niveau, sur leurs preuves.

## Exercices faciles

**E1 — Remplacer une décision.**
Six mois après l'ADR 0001, le contexte change : les visites des pages partenaires sont désormais **facturées**
aux partenaires, à la visite. Une visite comptée doit être une visite écrite en base, même si l'instance est
tuée juste après. Écrire l'ADR 0002, **au statut proposé** :
- ce qui change dans le contexte ;
- les critères, reclassés ;
- les options (au moins trois), avec ce qu'on sait déjà de chacune et ce qu'il reste à mesurer ;
- la modification à apporter à l'ADR 0001.

L'exercice intermédiaire fera les mesures et terminera l'ADR.

**E2 — Réversible ou non ?**
Pour chacune de ces décisions, estimer le coût d'un retour en arrière (qui est touché, quel travail), et en
déduire l'effort qu'elle mérite (décider en une heure, prototype, ADR, relecture large) :
1. l'intervalle d'écriture du `BatchCounter` ;
2. le nom d'une colonne de la table `visits`, lue par l'équipe de statistiques ;
3. `pgx` ou `database/sql` dans un service ;
4. le format JSON des événements publiés sur une file, lus par trois autres équipes ;
5. un champ `id` en entier auto-incrémenté plutôt qu'en UUID, dans une API publique ;
6. passer d'un monolithe modulaire à deux services ;
7. la bibliothèque de journalisation (`slog` ou une autre) ;
8. le chiffrement des sauvegardes de la base.

**E3 — Relire une PR.**
Un collègue propose un cache devant le calcul du classement, qui coûte 40 ms en base :

```go
package top

import (
	"context"
	"sync"
	"time"
)

type Cache struct {
	mu   sync.RWMutex
	val  []string
	at   time.Time
	ttl  time.Duration
	load func(context.Context) ([]string, error)
}

func NewCache(ttl time.Duration, load func(context.Context) ([]string, error)) *Cache {
	return &Cache{ttl: ttl, load: load}
}

// Get renvoie le classement, recalculé au plus une fois par ttl.
func (c *Cache) Get(ctx context.Context) ([]string, error) {
	c.mu.RLock()
	if time.Since(c.at) < c.ttl {
		v := c.val
		c.mu.RUnlock()
		return v, nil
	}
	c.mu.RUnlock()
	v, err := c.load(ctx)
	if err != nil {
		return nil, err
	}
	c.mu.Lock()
	c.val, c.at = v, time.Now()
	c.mu.Unlock()
	return v, nil
}
```

Écrire la revue : un résumé, puis chaque commentaire, classé (bloquant, à discuter, détail), avec le scénario
qui casse et une proposition. Prouver **au moins trois** défauts par un test (indice : 100 requêtes simultanées
à l'expiration ; un appelant qui modifie le résultat ; une base en panne pendant une minute). Écrire ensuite
la version corrigée, et montrer que les tests passent.

**E4 — La matrice qui décide trop vite.**
Les options de l'ADR 0001, notées de 1 à 5 sur ses quatre critères :

| | Base partagée (poids 0,4) | Latence (0,3) | Capacité (0,2) | Simplicité (0,1) |
|---|---|---|---|---|
| A | 1 | 2 | 1 | 5 |
| C | 2 | 4 | 4 | 5 |
| B | 5 | 5 | 5 | 2 |

Écrire un programme qui calcule le total de chaque option, puis, pour chaque poids, **de combien il faut le
modifier** (les autres poids étant ajustés pour que la somme reste 1, en gardant leurs proportions) pour
changer l'option gagnante. Quel poids est le plus fragile ? Que conclure sur la confiance à accorder au total ?

**E5 — Recevoir une revue.**
Ta PR ajoute un point d'accès `GET /top`. Tu reçois ces commentaires. Écrire une réponse à chacun : corriger,
refuser avec un argument, ou demander une précision.
1. « Pourquoi pas Redis pour ce cache ? »
2. « [bloquant] `limit` vient de la requête sans borne : `?limit=100000000` alloue un tableau énorme. »
3. « Je préfère `fetchTop` à `loadTop`. »
4. « [bloquant] Il faut un test pour le cas où la base est en panne. »
5. « Ce code est trop compliqué. »
6. « [bloquant] Utilise un `sync.Map`, c'est plus rapide. »

---

## Exercice intermédiaire — Mesurer, puis décider

Terminer l'ADR 0002 de l'exercice E1, en mesurant deux options qui ne perdent aucune visite confirmée :

1. **A** : une écriture par visite, telle quelle (déjà mesurée dans le cours).
2. **G, une validation groupée côté application** : `Visit` ajoute la visite au lot en cours, puis **attend**
   que ce lot soit validé en base avant de rendre la main. Une goroutine écrit le lot toutes les 5 ms, ou dès
   qu'il atteint 500 visites. Si l'écriture échoue, **toutes** les visites du lot reçoivent l'erreur.
3. Écrire `GroupCounter` et ses tests :
   - aucune visite n'est confirmée sans être en base ;
   - une panne de la base renvoie une erreur à chaque visite du lot ;
   - `Close` attend les visites en cours.
4. Mesurer avec le banc du cours (une option `-mode group`) : p50, p99, transactions validées, à 1 000, 5 000 et
   20 000 visites/s.
5. Terminer l'ADR 0002 : décision, conséquences, réexamen. Qu'est-ce que G coûte par rapport à B ?

---

## Défi — Relire une conception

Une équipe propose ce document. Rédiger la revue de conception : les questions à poser, les risques classés par
gravité, les informations manquantes, et une recommandation argumentée. Si la recommandation est une autre
option, l'écrire sous forme d'ADR au statut proposé.

> **Proposition : découper la plateforme en microservices**
>
> **Problème.** La plateforme (un monolithe Go, une base PostgreSQL) ralentit aux heures de pointe : le p99 des
> pages passe de 80 ms à 900 ms entre 12 h et 14 h. Les déploiements sont risqués : une modification du
> catalogue a cassé le paiement le mois dernier.
>
> **Proposition.** Découper le monolithe en quatre services (catalogue, commandes, paiement, comptes), chacun
> avec son équipe et son déploiement. Ils communiquent en HTTP. Chaque service garde pour l'instant ses
> tables dans la base actuelle ; on séparera les bases plus tard.
>
> **Bénéfices attendus.** Les microservices passent à l'échelle horizontalement : on ajoutera des instances
> du catalogue aux heures de pointe. Les déploiements seront indépendants. Les microservices sont la norme
> dans l'industrie.
>
> **Plan.** Les quatre services en un trimestre, puis bascule complète un dimanche soir. L'équipe compte
> quatre développeurs.
>
> **Commande.** Pour passer une commande, le service commandes appelle le catalogue (prix et stock), puis le
> paiement, puis décrémente le stock via le catalogue.

---

## Quiz

1. Que contient un ADR, et pourquoi le contexte est-il la partie la plus importante ?
2. Pourquoi ne modifie-t-on pas un ADR accepté ?
3. Comment doser l'effort consacré à une décision ?
4. Pourquoi classer les critères avant de comparer les options ?
5. Pourquoi inclure « ne rien faire » et l'option ennuyeuse ?
6. Quel est le piège d'une matrice pondérée ?
7. Dans quel ordre un relecteur senior regarde-t-il une PR ?
8. Pourquoi classer chaque commentaire de revue ?
9. Que faire d'un commentaire de revue avec lequel on n'est pas d'accord ?
10. Quels défauts chercher en priorité dans une PR Go qui ajoute une goroutine et un verrou ?
