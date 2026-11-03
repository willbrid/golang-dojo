# Module 1 — Exercices

Code dans `mes-solutions/niveau-13/module-01/`. Pour chaque composant : l'état et les invariants **d'abord**,
en commentaire, puis le code, puis le test qui vérifie les invariants. Un exercice rendu sans invariants
écrits est incomplet.

## Exercices faciles

**E1 — Fermer le pool.**
Ajouter `Close()`. Il détruit les ressources libres, fait échouer immédiatement les attendants (`ErrClosed`)
et tous les `Get` suivants, et fait détruire par `Put` les ressources rendues après la fermeture. Écrire
les nouveaux invariants (que deviennent I1 et I3 après `Close` ?). Quand `Close` peut-il rendre la main en
garantissant que **toutes** les ressources ont été détruites ?

**E2 — Une durée de vie maximale.**
Une ressource ne doit pas être prêtée plus de `MaxLifetime` après sa création (comme
`sql.DB.SetConnMaxLifetime`, N8 L5). Où mémoriser la date de création ? Qui détruit une ressource trop
vieille : `Get` qui la tire d'`idle`, `Put` qui la reçoit, ou une goroutine de nettoyage ? Quel invariant
ajoutez-vous, et quand est-il vérifiable ?

**E3 — Le pool en canaux seulement.**
Écrire un pool à base d'un canal bufferisé de `max` ressources (ou de jetons). Lesquels des invariants I1 à
I5 et des propriétés P1, P2 peut-il garantir ? Mesurer l'**équité** des deux pools : sous forte contention,
la distribution des temps d'attente (p50, p99, maximum), avec 64 emprunteurs et 4 ressources.

**E4 — Trouver le défaut.**
Ce loquet doit bloquer `Wait` tant qu'il est fermé. `Open` le libère, `Reset` le referme.

```go
type Latch struct {
	mu   sync.Mutex
	cond *sync.Cond
	open bool
}

func NewLatch() *Latch { l := &Latch{}; l.cond = sync.NewCond(&l.mu); return l }

func (l *Latch) Wait() {
	l.mu.Lock()
	if !l.open {
		l.cond.Wait()
	}
	l.mu.Unlock()
}

func (l *Latch) Open()  { l.mu.Lock(); l.open = true; l.cond.Broadcast(); l.mu.Unlock() }
func (l *Latch) Reset() { l.mu.Lock(); l.open = false; l.mu.Unlock() }
```

Le détecteur de races ne dit rien. Écrire l'invariant que `Wait` doit garantir à son retour, trouver
l'entrelacement qui le viole, écrire un test qui le montre (même rarement), et corriger.

**E5 — Vérifier l'équité.**
Instrumenter le pool du cours (dans le test seulement) pour enregistrer l'ordre d'arrivée des attendants et
l'ordre dans lequel ils sont servis. Vérifier P2 sous contrainte. Que se passe-t-il pour P2 quand un attendant
est annulé ?

---

## Exercice intermédiaire — Un regroupeur borné

Un composant `Batcher` regroupe des éléments pour les écrire par lots (comme les visites du projet 8) :

- `Add(ctx, item) error` ajoute un élément. Il **bloque** si `Capacity` éléments attendent déjà (pas de file
  sans borne), et échoue si `ctx` est annulé ;
- un lot part dès que `BatchSize` éléments sont prêts, ou `Interval` après le premier élément du lot en
  attente ;
- `flush(ctx, batch) error` est fourni par l'utilisateur. Il n'est jamais appelé deux fois en même temps, et
  un lot en échec est réessayé (N10 L9) ;
- `Close(ctx)` vide tout ce qui reste, puis rend la main.

1. Écrire l'état, les invariants (dont l'invariant **extérieur** : chaque élément accepté est écrit
   exactement une fois, même après des échecs de `flush`) et les propriétés de progrès.
2. Implémenter, avec une justification en commentaire de chaque endroit délicat.
3. Tester :
   - un stress de 16 producteurs sous `-race`, avec des `flush` qui échouent au hasard ;
   - la vérification au repos de l'invariant extérieur ;
   - `synctest` pour le déclenchement par `Interval` ;
   - un `Close` pendant que des `Add` sont bloqués.

---

## Défi — Vérifier la linéarisabilité

Une structure concurrente est **linéarisable** si chaque opération semble se produire instantanément à un
moment situé entre son appel et son retour. Toute exécution équivaut alors à une exécution séquentielle qui
respecte l'ordre réel des opérations qui ne se chevauchent pas.

1. Écrire un enregistreur d'**historique** : pour chaque opération `Put(k, v)` ou `Get(k)` sur un registre
   clé-valeur, l'instant d'appel, l'instant de retour et le résultat (horloge monotone).
2. Écrire un **vérificateur** qui, pour un historique court (une dizaine d'opérations par clé), cherche un
   ordre séquentiel valide. Il faut que cet ordre respecte la précédence réelle (si A rend la main avant que B
   ne commence, A précède B), et que chaque `Get` y lise la dernière valeur écrite. La recherche peut être
   exhaustive, avec retour arrière.
3. Le vérifier sur une implémentation correcte (une map sous `sync.Mutex`), puis sur une implémentation
   **fautive**, par exemple un cache de lecture par goroutine qui peut renvoyer une valeur périmée. Le
   vérificateur doit trouver des historiques non linéarisables pour la seconde, et jamais pour la première.
4. Combien d'opérations par clé ton vérificateur supporte-t-il avant d'être trop lent ? Pourquoi la
   vérification est-elle coûteuse en général ?

---

## Quiz

1. Que faut-il écrire avant le code d'un composant concurrent ?
2. Qu'est-ce qu'un invariant, et à quels moments doit-il être vrai ?
3. Quelle différence entre une propriété de sûreté et une propriété de progrès ? Un exemple de chaque.
4. Citer trois relations « arrivé-avant » du modèle mémoire de Go.
5. Qu'est-ce que le transfert de propriété par un canal ?
6. Pourquoi l'envoi `w <- r` sous le verrou est-il sûr dans le pool ?
7. Quel entrelacement la version erronée oublie-t-elle ?
8. Pourquoi le détecteur de races ne voit-il pas ce défaut ?
9. Qu'est-ce qu'un invariant extérieur, et quand peut-on le vérifier ?
10. Quand préférer un verrou à un canal ?
