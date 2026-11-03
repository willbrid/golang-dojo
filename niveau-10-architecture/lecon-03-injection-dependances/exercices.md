# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-10/lecon-03/`, en partant du programme de l'exemple.

## Exercices faciles

**E1 — L'`init` qui ouvre la base.**
Déplacer l'ouverture du stockage dans une fonction `init()` qui remplit une variable globale `var store
*Store`. Que se passe-t-il quand `DATABASE_URL` est absente ? Quand s'exécute cet `init` lors d'un
`go test` ? Peut-on encore tester les trois démarrages de `Example_run` ?

**E2 — `log.Fatal` au milieu.**
Remplacer le `return fmt.Errorf("usage…")` de `run` par `log.Fatal("usage…")`. Lancer le programme avec un
seul argument : le message « stockage fermé » apparaît-il ? Pourquoi ? Qu'est-ce que cela signifie pour une
vraie base ou un fichier en cours d'écriture ?

**E3 — Une option de plus.**
Ajouter `WithClock(Clock)` au `Notifier` (pour horodater les messages). Combien d'appels existants faut-il
modifier ? Et si `NewNotifier` avait pris une struct de configuration ? Et des paramètres positionnels ?

**E4 — Le délai à zéro.**
Écrire la variante `NewNotifier(out io.Writer, cfg NotifierConfig)`. Comment distinguer « délai non
renseigné, donc 2 s par défaut » de « pas de délai du tout » ? Proposer deux solutions.

**E5 — Le graphe.**
Dessiner (sur papier ou en texte) le graphe de dépendances du programme, et l'ordre de construction et
de fermeture. Ajouter un `Cache` entre le service et le stockage : quelles lignes changent, et où ?

---

## Exercice intermédiaire — Un vrai `main` de service

Écrire le `main` du service de prêts des leçons 1 et 2, en HTTP :

1. `run(ctx, args, getenv, stdout, stderr) error` lit et **valide** la configuration (adresse d'écoute,
   délais, type de stockage `memory` ou `postgres`), avec toutes les erreurs ensemble (N9 L2).
2. Construit le stockage choisi, le service, le routeur, le serveur HTTP avec ses délais (N7 L9).
3. S'arrête proprement quand `ctx` est annulé (`signal.NotifyContext` dans `main`, N7 L7) : arrêt du
   serveur avec délai, **puis** fermeture du stockage.
4. Test : démarrer `run` dans une goroutine avec une adresse `127.0.0.1:0`… problème : comment le test
   connaît-il le port choisi ? Proposer une solution qui ne modifie pas la signature pour le seul besoin du
   test (indice : un paramètre `ready func(addr string)`, ou un `net.Listener` injecté).
5. Test : annuler le contexte pendant une requête lente, vérifier qu'elle se termine et que le stockage
   est fermé **après**.

---

## Défi — Construire sans se tromper d'ordre

Le graphe d'un vrai service : configuration → journal → base → *repositories* → client HTTP externe →
services → handlers → serveur ; certaines ressources doivent être fermées (base, client, serveur).

1. Écrire un petit type `Closers` qui enregistre des fonctions de fermeture au fil de la construction et
   les exécute dans l'ordre **inverse**, en rassemblant les erreurs (`errors.Join`), même si l'une échoue.
2. Si la construction échoue au milieu (le client externe ne se configure pas), tout ce qui a déjà été
   ouvert doit être fermé. Le tester.
3. Comparer avec une suite de `defer` dans `run` : qu'apporte `Closers`, et quand préférer les `defer` ?
4. Lire la documentation de `wire` (sans l'installer) : quels problèmes résout-il, et lesquels de ces
   problèmes as-tu rencontrés dans ce défi ?

---

## Quiz

1. Qu'est-ce que l'injection de dépendances en Go, concrètement ?
2. Qu'est-ce que la racine de composition, et où se trouve-t-elle ?
3. Pourquoi éviter les variables globales et les `init()` à effets de bord ?
4. Que reçoit la fonction `run`, et pourquoi ?
5. Pourquoi n'appeler `os.Exit` que dans `main` ?
6. Quand utiliser des options fonctionnelles plutôt qu'une struct de configuration ?
7. Dans quel ordre fermer les ressources, et comment le garantir ?
8. Que signale un constructeur avec beaucoup de paramètres ?
9. Que fait `wire`, et pourquoi s'en passe-t-on souvent ?
10. Quelles dépendances faut-il injecter, et lesquelles peut-on appeler directement ?
