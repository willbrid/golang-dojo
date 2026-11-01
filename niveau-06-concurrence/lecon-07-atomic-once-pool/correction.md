# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Moderniser

`go fix` (analyseur `atomictypes`) transforme `var n int64` + `atomic.AddInt64(&n, 1)` en
`var n atomic.Int64` + `n.Add(1)`. Le type est plus sûr parce qu'il rend **impossible** un accès non
atomique : avec un `int64` nu, rien n'empêche d'écrire `n++` ou `fmt.Println(n)` ailleurs — une race
que le compilateur accepte. Avec `atomic.Int64`, `n++` ne compile pas ; on ne peut passer que par les
méthodes. Le type garantit aussi l'**alignement** sur 64 bits, nécessaire aux opérations atomiques sur
les architectures 32 bits.

## E2 — Le plafond

Avec `Load` puis `Add`, on obtient **plus de 100** succès (101, 104…). `-race` ne dit **rien** :
chaque accès est atomique, il n'y a pas de data race au sens du détecteur. C'est une erreur de
logique, comme le « vérifier puis agir » de la leçon 6.

```go
func TryAcquire(n *atomic.Int64, max int64) bool {
	for {
		old := n.Load()
		if old >= max {
			return false
		}
		if n.CompareAndSwap(old, old+1) {
			return true
		}
	}
}
```

Exactement 100 succès. Le `CompareAndSwap` échoue si une autre goroutine a modifié `n` depuis notre
`Load` ; on relit et on recommence. Sous forte contention, la boucle peut tourner plusieurs fois —
c'est le prix d'un algorithme sans verrou.

## E3 — La configuration mélangée

`-race` signale l'écriture de `c.Greeting` (dans `Reload`) concurrente des lectures dans `Handle`.
`atomic.Pointer` ne protège que **le pointeur**, pas la valeur pointée. Si la valeur est modifiée en
place, un lecteur peut voir un `Greeting` nouveau avec un `MaxItems` ancien, ou même une chaîne en
cours d'écriture. L'immuabilité est ce qui permet de se passer de verrou : on ne lit jamais quelque
chose qui change.

## E4 — `OnceValues` et l'erreur mémorisée

Le second appel rend **la même erreur** que le premier, sans relire le fichier : `OnceValues` a
mémorisé le résultat, erreur comprise. Version qui réessaie :

```go
type lazyConfig struct {
	mu     sync.Mutex
	loaded bool
	cfg    *Config
}

func (l *lazyConfig) Get() (*Config, error) {
	l.mu.Lock()
	defer l.mu.Unlock()
	if l.loaded {
		return l.cfg, nil
	}
	cfg, err := loadFile("config.toml")
	if err != nil {
		return nil, err          // rien n'est mémorisé : le prochain appel réessaiera
	}
	l.cfg, l.loaded = cfg, true
	return cfg, nil
}
```

Le chargement se fait **sous** verrou, donc jamais deux à la fois — une exception assumée à la règle
« rien de lent sous verrou » : ici, les autres appelants **doivent** attendre le résultat.

## E5 — Le pool qui fuit

Sans `Reset`, le second appel rend `"Bonjour, Awa !Bonjour, Awa !"` : le buffer réutilisé contenait
encore la première sortie. Dans un serveur, la réponse d'un utilisateur pourrait contenir une partie
de la réponse d'un **autre** — une fuite de données personnelles, causée par une ligne oubliée. C'est
exactement le type de bug qu'un test de deux appels consécutifs détecte immédiatement.

## Exercice intermédiaire — Statistiques sans verrou

```go
type Stats struct {
	requests  atomic.Int64
	errors    atomic.Int64
	sumNanos  atomic.Int64
	maxNanos  atomic.Int64
}

func (s *Stats) Observe(status int, latency time.Duration) {
	s.requests.Add(1)
	if status >= 500 {
		s.errors.Add(1)
	}
	s.sumNanos.Add(int64(latency))
	for {
		old := s.maxNanos.Load()
		if int64(latency) <= old || s.maxNanos.CompareAndSwap(old, int64(latency)) {
			break
		}
	}
}

type Snapshot struct {
	Requests, Errors int64
	Max, Mean        time.Duration
}

// Snapshot lit les compteurs séparément : sous charge, les valeurs peuvent provenir
// d'instants légèrement différents (une requête comptée dans la somme mais pas encore
// dans le nombre). Écart borné par le nombre de requêtes en vol ; acceptable pour des
// statistiques, pas pour de la comptabilité.
func (s *Stats) Snapshot() Snapshot {
	n := s.requests.Load()
	snap := Snapshot{Requests: n, Errors: s.errors.Load(), Max: time.Duration(s.maxNanos.Load())}
	if n > 0 {
		snap.Mean = time.Duration(s.sumNanos.Load() / n)
	}
	return snap
}
```

- Le maximum : on ne tente le `CompareAndSwap` que si la nouvelle latence est plus grande ; s'il
  échoue, quelqu'un a changé le maximum, on relit.
- L'incohérence du `Snapshot` est **documentée** : c'est ce qui distingue un choix conscient d'un bug.
  Si un instantané cohérent est requis, il faut un `Mutex` autour de toutes les mises à jour.
- Benchmark typique : la version atomique est 2 à 4 fois plus rapide que la version `Mutex` sous
  forte contention ; mais quatre opérations atomiques sur la même ligne de cache se disputent aussi
  (niveau 11 : *false sharing*), et l'écart se réduit quand on ajoute des champs.
- Sous `-race`, les totaux sont exacts : chaque `Add` est atomique.

## Défi — Quand le pool ne sert à rien

1. Pour un `bytes.Buffer` vide utilisé localement et jamais renvoyé, le compilateur peut le placer
   sur la **pile** : l'allocation directe ne coûte presque rien, et le pool ajoute l'assertion de
   type, le `Get` et le `Put` — il devient **plus lent**.
2. Pour des buffers de 64 Kio remplis à chaque requête, le pool évite l'allocation **et** la remise à
   zéro de 64 Kio : gain net, souvent d'un facteur 2 ou plus, et `B/op` s'effondre.
3. Sans la règle de taille, le 1 % de requêtes à 10 Mio laisse des buffers géants dans le pool : la
   mémoire du processus reste haute et chaque requête ordinaire qui récupère un de ces buffers garde
   10 Mio occupés pour écrire 2 Kio. Avec `if b.Cap() <= 1<<20 { pool.Put(b) }`, les géants sont
   abandonnés au ramasse-miettes.
4. Un pool vaut sa complexité quand : l'objet est **gros ou coûteux** à initialiser ; il est alloué
   **très souvent** sur un chemin chaud ; le profil mémoire (niveau 11) montre que ces allocations
   comptent ; et la réinitialisation est simple et fiable. Sinon, laisser faire l'allocateur.

## Quiz

1. Une opération indivisible : aucune autre goroutine ne peut en observer un état intermédiaire.
2. Une autre goroutine peut modifier `n` entre le `Load` et le `Add` ; le plafond est dépassé.
3. Elle remplace la valeur seulement si elle vaut encore l'ancienne valeur attendue ; on l'utilise en
   boucle (relire, calculer, tenter).
4. Le type interdit tout accès non atomique et garantit l'alignement.
5. La valeur pointée n'est jamais modifiée après publication : on en publie une nouvelle.
6. Que la fonction s'exécute une seule fois, au premier appel, que les appelants concurrents
   attendent sa fin, et que son résultat leur est visible.
7. Elle est mémorisée : tous les appels suivants la rendent, sans réessayer.
8. Il contient l'état de son utilisation précédente (données d'une autre requête).
9. Le ramasse-miettes peut le vider à tout moment ; ce n'est pas un stock d'objets à gérer.
10. Dès que l'invariant porte sur plusieurs variables ou que la logique dépasse un compteur, un
    drapeau ou un pointeur publié.
