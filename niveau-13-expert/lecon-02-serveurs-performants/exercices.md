# Module 2 — Exercices

Code dans `mes-solutions/niveau-13/module-02/`. Chaque réponse est une **mesure**, avec la commande qui la
reproduit.

## Exercices faciles

**E1 — Moins de mémoire par connexion.**
Le serveur de `conns` garde un `bufio.Reader` de 4 Kio par connexion, même inactive. Le remplacer par un
tampon **emprunté** à un `sync.Pool` seulement le temps de lire une ligne (on lit d'abord dans un petit
tableau sur la pile, puis on emprunte si la ligne est plus longue). Mesurer la mémoire par connexion pour
10 000 connexions inactives. Qu'est-ce qui reste, et peut-on descendre plus bas ?

**E2 — Échapper sans allouer.**
Écrire `appendJSONString(dst []byte, s string) []byte` sans allocation. Elle doit produire **exactement**
ce que produit `json.Marshal(s)` :
- `"` et `\` échappés ;
- les caractères de contrôle en `\u00XX` (sauf `\n`, `\r`, `\t`) ;
- `<`, `>` et `&` en `<`… ;
- U+2028 et U+2029 en ` ` et ` ` ;
- l'UTF-8 invalide remplacé par `�`.

La vérifier par un test de fuzzing (N5 L7) contre `json.Marshal`, puis ramener le budget de `Pooled` à 0.

**E3 — Un middleware qui ne casse pas `sendfile`.**
Écrire un middleware qui compte les octets envoyés par chaque réponse (pour une métrique). Servir un fichier
de 512 Mio par `http.ServeContent` à travers ce middleware, puis sans. Avec `strace -f -c -e
trace=sendfile,write`, vérifier si `sendfile` est encore utilisé. Le rendre transparent (une méthode
`ReadFrom` qui délègue, ou `Unwrap`), et mesurer à nouveau.

**E4 — Quand le *runtime* crée des fils.**
Lancer 200 goroutines qui dorment 1 s avec `time.Sleep`, puis 200 goroutines qui dorment 1 s par un appel
système bloquant (`syscall.Nanosleep`). Mesurer le nombre de fils du processus (`/proc/self/status`) pendant
chaque expérience. Expliquer la différence. Que se passerait-il avec 10 000 lectures de fichiers sur un disque
réseau lent ?

**E5 — Le prix d'une connexion neuve.**
Avec `server` et `client` du cours, comparer 20 000 requêtes en connexion persistante puis avec
`DisableKeepAlives: true` côté client. Mesurer le débit, et les allocations du serveur par requête. Combien
coûte une connexion TCP neuve (en local, sans TLS) ? Qu'ajouterait TLS ?

---

## Exercice intermédiaire — Un serveur clé-valeur sur TCP

Un protocole texte, une commande par ligne :

```
SET clé valeur\n   →  OK\n
GET clé\n          →  VALUE valeur\n   ou   NOTFOUND\n
```

1. Un serveur avec une goroutine par connexion, qui accepte le **pipelining** : un client envoie 100
   commandes d'un coup, puis lit les 100 réponses. Les réponses sont écrites dans un `bufio.Writer` et
   vidées quand il n'y a plus rien à lire.
2. Un budget : **0 allocation** par `GET` sur le chemin chaud (lecture de la ligne, découpe, recherche,
   réponse), vérifié par un test. Indice : les clés de la map sont des `string`, et `m[string(b)]` n'alloue
   pas.
3. Mesurer le débit avec 1, 8 et 64 clients, avec et sans pipelining (profondeur 1 et 100). Comparer avec une
   version HTTP du même service (`GET /kv/{clé}`).
4. Profiler (N11 L4) la version à 64 clients : où passe le temps ? Que reste-t-il à gagner, et est-ce dans le
   code ou dans le noyau ?

---

## Défi — 100 000 connexions

1. Pousser `conns` à 100 000 connexions. Quelle limite rencontre-t-on d'abord : les descripteurs
   (`ulimit -n`), les ports éphémères (`/proc/sys/net/ipv4/ip_local_port_range`), la mémoire ?
2. Contourner la limite des ports. Les adresses `127.0.0.2`, `127.0.0.3`… sont toutes locales : un client
   peut se lier à l'une d'elles (`net.Dialer{LocalAddr: …}`).
3. Mesurer, à 100 000 connexions, la mémoire par connexion avec la version de E1, le nombre de fils, et le
   temps d'un aller-retour sur toutes les connexions.
4. Écrire en dix lignes ce qu'il faudrait pour un million de connexions sur une machine : mémoire, réglages du
   noyau, et ce qui changerait dans le code.

---

## Quiz

1. Que fait le *runtime* quand une goroutine lit sur une socket sans données ?
2. Pourquoi 10 000 connexions n'occupent-elles qu'une dizaine de fils ?
3. Qu'est-ce qui coûte de la mémoire pour chaque connexion ?
4. Dans quels cas le *runtime* crée-t-il des fils supplémentaires ?
5. Comment `io.Copy` déclenche-t-il `sendfile` ?
6. Trois choses qui empêchent `sendfile` ?
7. Qu'est-ce qu'un budget d'allocation, et comment le faire respecter ?
8. Pourquoi mesurer un handler avec un `ResponseWriter` minimal plutôt qu'avec `httptest.NewRecorder` ?
9. Combien d'allocations `net/http` fait-il par requête, et qu'en conclure ?
10. Quelle optimisation du cours donne le plus grand gain, et pourquoi ?
