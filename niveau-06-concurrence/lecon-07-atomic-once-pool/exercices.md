# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-06/lecon-07/`. Tout avec `-race`.

## Exercices faciles

**E1 — Moderniser.**
Écrire un compteur avec l'ancienne API (`var n int64` et `atomic.AddInt64(&n, 1)`,
`atomic.LoadInt64(&n)`). Lancer `go fix -diff`. Que propose `atomictypes` ? Pourquoi le type
`atomic.Int64` est-il plus sûr que la fonction sur `*int64` ? *(Penser à un accès non atomique
oublié.)*

**E2 — Le plafond.**
Écrire `TryAcquire(n *atomic.Int64, max int64) bool` qui incrémente `n` seulement s'il est
inférieur à `max`. D'abord avec `Load` puis `Add` : lancer 1 000 goroutines avec `max = 100` et
compter les succès (plusieurs essais). Puis avec une boucle `CompareAndSwap`. Que dit `-race` dans
le premier cas ?

**E3 — La configuration mélangée.**
Modifier l'exemple pour que `Reload` modifie **en place** les champs de la config publiée
(`c := s.cfg.Load(); c.Greeting = "Salut"`). Que dit `-race` ? Pourquoi la règle « immuable une fois
publiée » est-elle indispensable ?

**E4 — `OnceValues` et l'erreur mémorisée.**
Écrire un chargement de configuration avec `sync.OnceValues` qui lit un fichier. Appeler la fonction
alors que le fichier n'existe pas, créer le fichier, rappeler. Que se passe-t-il ? Proposer une
version qui réessaie après un échec mais n'exécute jamais deux chargements en même temps (indice :
un `Mutex` et un champ « déjà chargé »).

**E5 — Le pool qui fuit.**
Dans l'exemple, retirer `b.Reset()`. Écrire un test qui appelle `render` deux fois de suite et
vérifie le second résultat. Que contient-il ? Imaginer la conséquence dans un serveur où le buffer
contiendrait la réponse d'un autre utilisateur.

---

## Exercice intermédiaire — Statistiques de serveur sans verrou

Écrire un type `Stats` pour un serveur à fort trafic :

```go
func (s *Stats) Observe(status int, latency time.Duration)
func (s *Stats) Snapshot() Snapshot   // requêtes, erreurs (status >= 500), latence max, latence moyenne
```

**Contraintes :**
1. `Observe` est appelé par des milliers de goroutines : **aucun verrou**, seulement des atomiques.
2. La latence **maximale** se met à jour avec une boucle `CompareAndSwap`.
3. La moyenne se calcule à partir d'une somme et d'un compte atomiques. Le `Snapshot` lit plusieurs
   atomiques **séparément** : quelle incohérence peut-il observer (par exemple une somme qui inclut
   une requête que le compte n'inclut pas encore) ? Est-ce acceptable pour des statistiques ? Le
   documenter.
4. Benchmark parallèle de `Observe` (`b.RunParallel`) contre une version avec `Mutex`. Écart ?
5. Test sous `-race` avec 100 goroutines × 10 000 observations : les totaux sont-ils exacts ?

---

## Défi — Quand le pool ne sert à rien

1. Écrire un benchmark où un `sync.Pool` de `*bytes.Buffer` est **plus lent** que l'allocation
   directe. *(Indice : un objet minuscule, que le compilateur pourrait même placer sur la pile —
   niveau 11, leçon 2.)*
2. Écrire un benchmark où il est **nettement** plus rapide (gros buffers, par exemple 64 Kio).
3. Ajouter au pool la règle « ne pas remettre un buffer de plus de 1 Mio ». Mesurer l'effet sur un
   flux de requêtes dont 1 % produit 10 Mio de sortie.
4. Conclusion en cinq lignes : à quelles conditions un `sync.Pool` vaut-il sa complexité ?

---

## Quiz

1. Qu'est-ce qu'une opération atomique ?
2. Pourquoi `if n.Load() < max { n.Add(1) }` est-il incorrect ?
3. Que fait `CompareAndSwap`, et comment l'utilise-t-on ?
4. Pourquoi préférer `atomic.Int64` aux fonctions `atomic.AddInt64(&n, …)` ?
5. Quelle règle rend `atomic.Pointer[Config]` correct pour une configuration rechargeable ?
6. Que garantit `sync.OnceValue` ?
7. Que devient une erreur rendue par la fonction passée à `OnceValues` ?
8. Pourquoi faut-il réinitialiser un objet pris dans un `sync.Pool` ?
9. Pourquoi un `sync.Pool` ne convient-il pas pour des connexions ?
10. Quand préférer un `Mutex` à un atomique ?
