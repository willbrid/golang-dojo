# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — La chaîne coupée

Le test échoue au bout de 2 secondes (« la base n'a jamais vu l'annulation ») : le client est parti
après 100 ms, mais la « base » continue ses 10 secondes de travail — sous charge, des milliers de
requêtes SQL pour des clients absents. Un seul mot (`Background` au lieu de `r.Context()`) suffit à
casser toute la chaîne.

## E2 — Observer l'abandon

```go
func slow(log io.Writer) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		select {
		case <-time.After(5 * time.Second):
			fmt.Fprintln(w, "fini")
		case <-r.Context().Done():
			fmt.Fprintln(log, "abandon :", context.Cause(r.Context()))
		}
	}
}
```

Avec `curl --max-time 1`, le journal affiche `abandon : context canceled` : le serveur voit la connexion
fermée par le client et annule le contexte, sans cause plus précise.

## E3 — Le budget partagé

Les deux premières dépendances réussissent (800 ms) ; la troisième n'a plus que 200 ms et est
interrompue avec `context.DeadlineExceeded` (enveloppé). Le client reçoit **504**. Personne n'a calculé
« 1 s − 800 ms » : le contexte porte une **échéance absolue**, que chaque appel respecte.

## E4 — Le mauvais contexte d'arrêt

`ctx` est **déjà annulé** au moment où `Shutdown` est appelé : `Shutdown` cesse d'accepter, ferme les
connexions inactives, puis rend **immédiatement** `context.Canceled` sans attendre les requêtes en
cours. Selon le moment, la requête en cours peut encore se terminer (le serveur ne la coupe pas tout de
suite) mais rien ne garantit plus qu'on l'attend : `Run` rend une erreur et le programme sort, coupant
la requête. Le test devient instable — symptôme typique.

## E5 — Les goroutines de fond

```go
var wg sync.WaitGroup
wg.Go(func() { purgeLoop(ctx, cache) })   // boucle de service avec <-ctx.Done() (N6 L4)
err := report.Run(ctx, srv, 10*time.Second)
wg.Wait()                                   // attendre la fin de la purge AVANT de sortir
```

Le même contexte arrête le serveur **et** la purge ; `wg.Wait()` garantit qu'aucune goroutine ne
travaille encore quand `main` rend la main. `NumGoroutine()` revient au niveau de départ.

## Exercice intermédiaire — Une passerelle

```go
func (g *Gateway) dashboard(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 800*time.Millisecond)
	defer cancel()

	var profile Profile
	var orders []Order
	degraded := false

	eg, egCtx := errgroup.WithContext(ctx)
	eg.Go(func() error {
		var err error
		profile, err = g.fetchProfile(egCtx)
		return err                                   // obligatoire : un échec fait échouer la requête
	})
	eg.Go(func() error {
		octx, ocancel := context.WithTimeout(egCtx, 300*time.Millisecond)   // budget propre, plus court
		defer ocancel()
		var err error
		orders, err = g.fetchOrders(octx)
		if errors.Is(err, context.DeadlineExceeded) && egCtx.Err() == nil {
			degraded, orders = true, nil             // facultatif : on dégrade au lieu d'échouer
			return nil
		}
		return err
	})
	if err := eg.Wait(); err != nil {
		if errors.Is(err, context.Canceled) && r.Context().Err() != nil {
			return                                   // client parti
		}
		writeProblem(w, err)                         // leçon 5 ; 504 si DeadlineExceeded
		return
	}
	writeJSON(w, http.StatusOK, Dashboard{Profile: profile, Orders: orders, Degraded: degraded})
}
```

- `egCtx.Err() == nil` distingue « `orders` a dépassé **son** budget » (on dégrade) de « tout est annulé
  » (client parti, `profile` en échec) — où il faut propager.
- Les variables `profile`, `orders`, `degraded` sont écrites par une seule goroutine chacune et lues
  après `Wait` : pas de verrou (N6 L5).
- La propagation vers les services distants se vérifie comme dans le cours : chaque faux service
  signale sur un channel l'annulation de son `r.Context()`.

## Défi — Arrêt en deux temps

```go
var ready atomic.Bool
ready.Store(true)
mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
	if !ready.Load() {
		http.Error(w, "arrêt en cours", http.StatusServiceUnavailable)
		return
	}
	io.WriteString(w, "ok")
})

<-ctx.Done()                                   // signal reçu
ready.Store(false)                             // 1. ne plus être choisi par le répartiteur
time.Sleep(drainDelay)                         // 2. continuer à servir pendant qu'il nous retire
shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
defer cancel()
srv.Shutdown(shutdownCtx)                      // 3. arrêt gracieux
```

Si l'on appelait `Shutdown` immédiatement, le répartiteur — qui ne vérifie la disponibilité que toutes
les quelques secondes — continuerait d'envoyer du trafic vers un serveur qui **refuse les connexions** :
les clients verraient des erreurs `connection refused` pendant cette fenêtre, à chaque déploiement. Le
délai de drainage (`time.Sleep` assumé ici : c'est une attente de **temps réel** voulue, pas une
synchronisation) fait disparaître ces erreurs.

## Quiz

1. Quand le client ferme la connexion (ou annule le flux HTTP/2) et quand le handler se termine.
2. Parce que la chaîne d'annulation serait coupée : l'abandon du client ne se propagerait plus.
3. `ctx, cancel := context.WithTimeout(r.Context(), d)` puis `defer cancel()`.
4. Le temps restant diminue d'autant : l'échéance est absolue, partagée par tous les appels.
5. `DeadlineExceeded` : 504 (ou 503) ; `Canceled` : ne rien répondre, le client est parti.
6. Il cesse d'accepter, ferme les connexions inactives et attend la fin des requêtes en cours, dans la
   limite du contexte fourni.
7. Parce que le contexte de signal est déjà annulé : `Shutdown` n'attendrait rien.
8. `http.ErrServerClosed`.
9. `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)`.
10. Les connexions détournées (WebSockets) et les goroutines de fond lancées par le programme.
