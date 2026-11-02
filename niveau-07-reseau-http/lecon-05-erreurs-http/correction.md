# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — La fuite

Le client reçoit `"detail": "pq: connection refused to 10.0.3.7:5432"`. Ce que l'attaquant en tire :
le **type de base** (PostgreSQL, via le préfixe `pq:` d'un pilote connu — donc les vulnérabilités à
essayer) ; une **adresse réseau interne** et un port (cartographie du réseau, cible d'une attaque SSRF) ;
le fait que la base est **en panne** (moment propice à d'autres attaques, ou confirmation qu'une attaque
de déni de service fonctionne). Sur d'autres erreurs : noms de tables, fragments de requêtes SQL,
chemins de fichiers, versions.

## E2 — Comparer des messages

Après le changement de message, le test « introuvable » **casse** — bon signal, mais pour une mauvaise
raison (on a changé un texte, pas un comportement). Pire : une erreur **différente** dont le texte
contient « introuvable » (« fichier de configuration introuvable », une erreur interne) devient un 404
au lieu d'un 500. Aucun test ne le voit. Le texte d'une erreur est fait pour les humains ; le contrat,
c'est la sentinelle ou le type.

## E3 — L'identifiant de requête

```go
p := toProblem(err)
p.RequestID = mw.RequestID(r.Context())          // extension `request_id`
fmt.Fprintf(log, "%s %s %s : %v\n", p.RequestID, r.Method, r.URL.Path, err)
```

L'identifiant ne révèle rien du système (c'est un nombre aléatoire) mais permet au support de retrouver
**la** ligne de journal correspondant à la plainte d'un client. C'est le pont entre la réponse
générique et le détail interne.

## E4 — 404 ou 403

- **403** : honnête, et utile quand l'existence n'est pas un secret (« ce document existe, demandez
  l'accès à son propriétaire »).
- **404** : quand l'existence elle-même est une information (un compte bancaire, un dossier médical) —
  sinon un attaquant **énumère** les identifiants valides en observant 403 contre 404.

Pour une API où les identifiants sont séquentiels, 404 est presque toujours le bon choix.

## E5 — Erreur après écriture

Le client reçoit un **200** avec un JSON **tronqué** — puis le corps du *problem* concaténé — et le
serveur journalise `http: superfluous response.WriteHeader call`. Le client ne peut pas savoir proprement
que la réponse est fausse. Avec un `bytes.Buffer` : on encode tout, et seulement si l'encodage réussit,
on écrit statut + buffer ; en cas d'erreur, on rend l'erreur sans avoir rien écrit. Coût : toute la
réponse en mémoire. Pour de très grosses réponses, on revient au streaming (leçon 2) avec un trailer ou
une ligne de contrôle pour signaler une fin anormale.

## Exercice intermédiaire — Un modèle d'erreur pour la bibliothèque

```go
var mapping = []struct {
	err    error
	status int
	typ    string
}{
	{library.ErrBookNotFound, http.StatusNotFound, "/problems/book-not-found"},
	{library.ErrMemberNotFound, http.StatusNotFound, "/problems/member-not-found"},
	{library.ErrAlreadyLoaned, http.StatusConflict, "/problems/already-loaned"},
	{library.ErrNotLoaned, http.StatusConflict, "/problems/not-loaned"},
}

func toProblem(err error) Problem {
	var verr *library.ValidationError
	if errors.As(err, &verr) {                    // les TYPES à part : ils portent des données
		return Problem{Type: "/problems/validation", Status: 422, Fields: verr.Fields}
	}
	for _, m := range mapping {
		if errors.Is(err, m.err) {
			return Problem{Type: m.typ, Status: m.status, Title: m.err.Error()}
		}
	}
	return Problem{Type: "/problems/internal", Status: 500, Title: "Erreur interne"}
}
```

- Les types d'erreur (`errors.As`) ne tiennent pas dans une table de valeurs : un `switch` ou une liste
  de fonctions de correspondance à part.
- Utiliser `m.err.Error()` comme `title` suppose que le message de la sentinelle est **fait pour le
  client** (court, sans donnée) : c'est le cas des sentinelles, pas des erreurs enveloppées.
- **Test d'exhaustivité** : le domaine exporte pour les tests `var AllErrors = []error{ErrBookNotFound,
  …}` (dans un `export_test.go`, N5 L1, ou un fichier ordinaire), et le test vérifie que chaque élément
  produit un statut **différent de 500**. Ajouter une sentinelle sans l'ajouter à la table fait échouer
  le test — à condition d'avoir aussi mis à jour `AllErrors`, ce qu'une revue doit vérifier.
- **404 et 405 du routeur** : ils sont écrits par `ServeMux` lui-même. Deux solutions : enregistrer un
  motif « attrape-tout » `"/"` qui rend un problem 404 (mais le 405 automatique reste en texte) ; ou un
  middleware autour du routeur qui intercepte les réponses 404/405 **en texte** (via un
  `ResponseWriter` enveloppé qui regarde le `Content-Type` au `WriteHeader`) et les remplace par un
  problem. La seconde est plus complète et plus délicate à écrire.

## Défi — Des erreurs pour les humains et pour les machines

1. Un `code` court et stable sert de **clé de traduction** : l'application mobile a son propre fichier
   de traductions indexé par `INSUFFICIENT_FUNDS`, et affiche un message dans la langue de l'utilisateur
   sans dépendre du texte du serveur.
2. Une table `map[string]map[string]string` (`type` → langue → titre) côté HTTP, et
   `r.Header.Get("Accept-Language")` analysé simplement (premier code de langue reconnu, sinon `fr`). Le
   domaine ne change pas : la langue est une question de **présentation**. Pour une analyse correcte de
   l'en-tête (qualités `q=0.8`), voir `golang.org/x/text/language`.
3. Pointeurs JSON : `/items/3/quantity` désigne le champ `quantity` du 4ᵉ élément de `items`. La
   validation construit la clé avec `fmt.Sprintf("/items/%d/quantity", i)` ; le client sait exactement
   quelle ligne du formulaire surligner.

## Quiz

1. Pour rester réutilisable (CLI, gRPC, tâches) et testable sans HTTP.
2. Pour la cohérence : un même cas donne toujours le même statut et le même format.
3. Un 500 générique, sans détail.
4. Les erreurs remontent naturellement (`return err`) ; plus de `return` oublié ; format uniforme.
5. `type`, `title`, `status`, `detail`, `instance`, plus des extensions ; le `type`.
6. `application/problem+json`.
7. Textes d'erreurs internes, requêtes SQL, chemins, adresses, versions, piles.
8. Non en erreur : ce sont des erreurs du client ; une métrique suffit.
9. Quand l'existence même de la ressource est une information sensible.
10. Parce qu'après le premier octet, le statut est parti : la réponse d'erreur serait incohérente.
