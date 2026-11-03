# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-12/lecon-05/`, en partant du paquet `config` de l'exemple.

## Exercices faciles

**E1 — Un réglage de plus.**
Ajouter `REQUEST_TIMEOUT` / `-request-timeout` (durée, défaut 5 s, entre 100 ms et 1 min), avec ses tests :
défaut, environnement, option, valeur invalide, hors plage. Combien de lignes faut-il toucher ? Qu'est-ce
qui pourrait être factorisé ?

**E2 — Un préfixe.**
Faire lire toutes les variables avec un préfixe : `SVC_ADDR`, `SVC_WORKERS`… (le préfixe est un paramètre de
`Load`). Pourquoi un préfixe évite-t-il des surprises ? Trouver une variable courante sur ta machine ou dans
un conteneur qui entrerait en conflit avec un nom sans préfixe.

**E3 — La faute de frappe.**
`SVC_WORKRES=8` est ignorée en silence, et le service démarre avec 4 workers. Faire signaler par `Load`
toute variable qui commence par le préfixe et qu'il ne connaît pas. Faut-il en faire une erreur ou un
avertissement ? Justifier.

**E4 — Des secrets en fichier, partout.**
Écrire `secret(env Env, name string) (string, error)`, qui lit `NAME` ou `NAME_FILE` (pas les deux), et
l'utiliser pour `DATABASE_URL` et un nouveau `API_TOKEN`. Ajouter un type `Secret` dont la méthode `String()`
renvoie `"[masqué]"`. Que donne alors `fmt.Printf("%v", cfg)` ? Et `%#v` ? Et `json.Marshal(cfg)` ?

**E5 — Ce que voit `docker inspect`.**
Lancer un conteneur avec `-e DATABASE_URL=postgres://app:s3cr3t@…`, puis avec un fichier monté et
`DATABASE_URL_FILE`. Que montre `docker inspect --format '{{.Config.Env}}'` dans chaque cas ? Et
`cat /proc/1/environ` depuis un autre conteneur qui partage l'espace de processus (`--pid container:…`) ?

---

## Exercice intermédiaire — Un fichier de configuration

Certaines équipes veulent un fichier, pour des réglages nombreux qui changent rarement.

1. Ajouter `CONFIG_FILE` : un fichier JSON lu **entre** les valeurs par défaut et l'environnement. La
   précédence devient : défauts < fichier < environnement < options.
2. Les champs inconnus du fichier sont une erreur (`json.Decoder.DisallowUnknownFields`, N4), comme les
   fautes de frappe de E3.
3. Les durées s'écrivent `"10s"` dans le fichier. Comment faire, puisque `time.Duration` est un entier en
   JSON ? (Indice : un type qui implémente `encoding.TextUnmarshaler`.)
4. Le fichier ne doit **jamais** contenir de secret : `Load` refuse un champ `database_url` dans le fichier.
5. Tester la précédence complète par une table : chaque ligne donne le fichier, l'environnement, les options
   et la valeur attendue.

---

## Défi — Changer le niveau de journal sans redémarrer

1. Le service journalise avec `slog`. Le niveau doit pouvoir changer à chaud pendant un incident.
   Utiliser un `slog.LevelVar` (son niveau se change à tout moment, sans course) dans le gestionnaire de
   journaux.
2. Sur `SIGHUP` (`signal.Notify`), le service relit **toute** la configuration avec `Load`. Si elle est
   valide, il applique le **seul** réglage modifiable à chaud (le niveau) et journalise le changement. Si
   elle est invalide, il garde l'ancienne et journalise l'erreur, sans s'arrêter.

   Que faire si un réglage non modifiable à chaud (l'adresse d'écoute) a changé ?
3. Publier la configuration courante par un `atomic.Pointer[config.Config]`, que lisent les handlers.
   Écrire un test, sous `-race`, où des goroutines lisent la configuration pendant qu'on la remplace.
4. Comparer avec une route d'administration `PUT /admin/log-level`. Avantages, risques, et sur quelle adresse
   l'exposer ?

---

## Quiz

1. Qu'est-ce qui est de la configuration, et qu'est-ce qui ne l'est pas ?
2. Quelle précédence entre valeurs par défaut, environnement et options ?
3. Pourquoi lire la configuration une seule fois, dans `main` ?
4. Pourquoi injecter `Getenv` plutôt que d'appeler `os.Getenv` ?
5. Pourquoi valider toute la configuration au démarrage ?
6. Pourquoi rapporter toutes les erreurs d'un coup ?
7. Où ne faut-il jamais mettre un secret ? Où le mettre ?
8. Qu'est-ce que la convention `NOM_FILE` ?
9. Comment une erreur peut-elle divulguer un secret ? Un exemple.
10. Comment changer un réglage sans redémarrer, sans course de données ?
