# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-11/lecon-04/`, en partant du paquet `words` de l'exemple.

## Exercices faciles

**E1 — Reproduire le diagnostic.**
Profiler `BenchmarkTopSlow`, puis lire successivement `top`, `top -cum` et `list TopSlow`. Quelles sont les
trois premières lignes de `top` ? Pourquoi ne permettent-elles pas, seules, de savoir quoi corriger ?

**E2 — Le *flamegraph*.**
Ouvrir `go tool pprof -http=:8080 words.test cpu.out`, menu *View → Flame Graph*. Repérer le rectangle de
`regexp.MustCompile`. Quelle largeur (en %) ? Retrouve-t-on le chiffre de `top -cum` ?

**E3 — Pas assez d'échantillons.**
Profiler `TopScan` avec `-benchtime=1x` (une seule itération, ≈ 36 ms). Combien d'échantillons ? Les
proportions sont-elles stables d'une exécution à l'autre ? Recommencer avec `-benchtime=3s`.

**E4 — Le poste suivant.**
Le profil de `TopScan` désigne `strings.ToLower` (≈ 60 %). Écrire `TopLower` qui supprime cette allocation
par mot (plusieurs idées : convertir le texte entier une seule fois ; ne convertir que si le mot contient une
majuscule ; une map indexée par la forme d'origine, convertie à la fin). Vérifier l'équivalence, mesurer,
reprofiler : quel est le nouveau premier poste ?

**E5 — PGO.**
Écrire un petit programme `cmd/top` qui lit un gros fichier texte et affiche les 10 mots les plus fréquents
avec `TopScan`. Le profiler (`runtime/pprof`) sur un fichier réaliste, copier le profil en
`cmd/top/default.pgo`, reconstruire, et comparer les durées (dix exécutions de chaque, test de rang de la
leçon 1). Le gain est-il démontré ?

---

## Exercice intermédiaire — Profiler un service

Le service de prêts (niveau 10) ou ton projet 3/4 :

1. Ajouter `net/http/pprof` sur une adresse d'administration distincte (`127.0.0.1:6060`), jamais sur
   l'adresse publique. Écrire le test qui le vérifie (l'adresse publique répond 404 à `/debug/pprof/`).
2. Lancer une charge (le générateur de la leçon 1) et collecter 20 s de profil
   (`go tool pprof http://127.0.0.1:6060/debug/pprof/profile?seconds=20`).
3. Dans `top -cum`, quelle part revient à ton code, à `encoding/json`, à `net/http`, au *runtime* (GC) ?
4. Choisir **un** poste, le corriger, prouver le gain par un benchmark **et** par une nouvelle mesure de
   bout en bout. Rédiger en cinq lignes ce que le profil a révélé que tu n'aurais pas deviné.

---

## Défi — Un profil sous concurrence

1. Paralléliser `TopScan` : découper le texte en N morceaux (aux fins de ligne), compter chaque morceau
   dans une goroutine avec sa propre map, fusionner à la fin.
2. Mesurer pour N = 1, 2, 4, 8 (`-cpu 1,2,4,8` et `GOMAXPROCS`). Le gain est-il linéaire ?
3. Profiler la version à 8 goroutines : quelle part pour la fusion des maps ? pour le GC ? Qu'est-ce que
   `Duration: 1s, Total samples = 6s (600%)` signifie dans l'en-tête ?
4. Comparer avec une seule map partagée protégée par un `sync.Mutex`. Que montre le profil CPU ? Que
   **ne** montre-t-il **pas** (indice : leçon 5) ?

---

## Quiz

1. Comment fonctionne un profil CPU ?
2. Trois façons de collecter un profil CPU ?
3. Quelle différence entre *flat* et *cum* ?
4. Pourquoi commencer par `top -cum` ?
5. Que montre `list` ?
6. Que représente la largeur d'un rectangle dans un *flamegraph* ?
7. Pourquoi ne corriger qu'une chose à la fois, et reprofiler ensuite ?
8. Pourquoi ne jamais exposer `/debug/pprof` publiquement ?
9. Qu'est-ce que le profil CPU ne montre pas ?
10. Qu'est-ce que PGO, et comment l'activer ?
