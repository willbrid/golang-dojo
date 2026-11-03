# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-12/lecon-06/`, en partant du paquet `logx` de l'exemple.

## Exercices faciles

**E1 — Réécrire en structuré.**
Transformer ces lignes en appels `slog` (message fixe, attributs typés), et dire quel niveau donner à
chacune :

```go
log.Printf("utilisateur %d connecté depuis %s", id, ip)
log.Printf("ERREUR : impossible de lire la commande %s : %v", cmdID, err)
log.Printf("cache : %d/%d entrées, taux de succès %.1f%%", n, max, rate)
log.Printf("relance %d/3 de l'appel au service de paiement", attempt)
```

Quelle donnée de la première ligne faut-il peut-être **ne pas** journaliser ?

**E2 — La source.**
Activer `AddSource: true` dans les options du gestionnaire. Que devient chaque ligne ? Combien coûte
l'option (benchmark) ? La ligne indiquée est-elle la bonne quand on journalise depuis une fonction
utilitaire `logError(log, err)` ? Lire la documentation de `slog.Record` et de `runtime.Callers` pour
comprendre comment corriger cela.

**E3 — Les groupes.**
Journaliser une requête sortante avec `slog.Group("http", "method", …, "status", …)`, puis avec
`log.WithGroup("http")`. Quelle différence dans le JSON produit ? Que donne `ReplaceAttr` pour un attribut
`token` placé dans le groupe : la fonction reçoit-elle le nom du groupe ?

**E4 — Le masquage qui ne suffit pas.**
Trouver **trois** façons de faire passer un secret dans un journal malgré `redact` (par exemple sous une
autre clé, dans un message, dans une erreur). Pour chacune, proposer la protection adaptée.

**E5 — Les journaux dans les tests.**
Un test qui échoue doit montrer les journaux du code testé, mais un test qui réussit ne doit pas les
afficher. Écrire un `io.Writer` qui envoie chaque ligne à `t.Log`, et l'utiliser avec `logx.New`. Quel
emplacement (`fichier:ligne`) `t.Log` affiche-t-il devant chaque ligne ? Que se passe-t-il si une goroutine
journalise **après** la fin d'un sous-test, puis après la fin d'un test de premier niveau ? Mesurer plutôt que
supposer. Comparer avec `t.Output()`, ajouté en Go 1.25.

---

## Exercice intermédiaire — Les journaux du service de prêts

Pour le service de prêts du niveau 10 (ou ton projet 4/7) :

1. Remplacer toute journalisation par `slog`, construite dans `main` avec `logx.New`. Format et niveau
   viennent de la configuration (leçon 5).
2. Le middleware de N10 L10 écrit sa ligne par requête avec `slog`. L'identifiant de requête et, si
   l'utilisateur est authentifié, son **identifiant** (pas son nom ni son e-mail) sont dans le contexte.
3. Chaque erreur est journalisée **une** fois. Repérer dans le code les endroits qui journalisent et
   renvoient la même erreur, et les corriger.
4. Ajouter `LogValue` sur les types qui contiennent des données personnelles ou des secrets (emprunteur,
   configuration).
5. Écrire un test qui fait une requête complète et vérifie, en relisant le JSON produit :
   - l'identifiant de requête est présent dans **toutes** les lignes de la requête ;
   - aucune ligne ne contient le mot de passe ni l'e-mail de l'utilisateur de test.

---

## Défi — Échantillonner et limiter

Un incident fait écrire 50 000 lignes d'erreur identiques par seconde : coût de stockage, collecteur
saturé, et les autres lignes sont noyées.

1. Écrire un gestionnaire `SampleHandler` qui enveloppe un `slog.Handler`. Pour chaque couple (niveau,
   message), il laisse passer les N premières lignes par seconde, puis une sur M. Il écrit chaque seconde une
   ligne de synthèse : « 49 872 lignes "lecture impossible" supprimées ».
2. Il doit être sûr en concurrence (`-race`), et respecter le contrat de `slogtest`. Attention à
   `WithAttrs` : les copies doivent-elles partager les compteurs ?
3. Mesurer son coût par ligne, comparé au gestionnaire nu, avec 8 goroutines (`RunParallel`).
4. Faut-il échantillonner les lignes `ERROR` ? Les lignes de requête `INFO` ? Où se situe la limite entre
   « économiser » et « perdre la preuve dont on aura besoin pendant l'incident » ? Proposer une règle.

---

## Quiz

1. Pourquoi un message fixe et des attributs, plutôt qu'une phrase formatée ?
2. Quelle différence entre les paires clé-valeur et `slog.Attr` ?
3. Quand utiliser chaque niveau ?
4. Pourquoi journaliser une erreur une seule fois, et où ?
5. Où un service doit-il écrire ses journaux ?
6. Comment l'identifiant de requête arrive-t-il dans chaque ligne ?
7. Pourquoi un gestionnaire enveloppant doit-il redéfinir `WithAttrs` et `WithGroup` ?
8. Quelles sont les deux protections contre les secrets dans les journaux ? Leurs limites ?
9. Que coûte une ligne `Debug` désactivée ? Qu'est-ce qui est payé malgré tout ?
10. Qu'est-ce que `testing/slogtest` ?
