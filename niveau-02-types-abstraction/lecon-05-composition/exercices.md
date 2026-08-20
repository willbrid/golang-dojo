# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-02/lecon-05/`.

## Exercices faciles

**E1 — Promotion.**
Créer `Base` avec un champ et une méthode, puis `Derived` qui l'embarque. Accéder au champ et
à la méthode de trois façons : promue, explicite via le nom du champ, et via une variable
intermédiaire. Puis tenter `var b Base = derived`. Relever le message d'erreur exact et
l'expliquer.

**E2 — Masquage et absence de dispatch virtuel.**
Reproduire l'exemple `Base.Describe()` / `Derived.Greet()` du cours. **Prédire la sortie avant
d'exécuter.** Puis écrire en trois phrases ce qu'un langage à héritage aurait affiché, et
pourquoi Go fait autrement.

**E3 — Ambiguïté.**
Créer `A` et `B` ayant chacun un champ `Name` et une méthode `Hello()`. Créer `C` qui embarque
les deux. Tenter `c.Name` et `c.Hello()`. Relever les messages. Corriger de deux façons.

**E4 — Composition d'interfaces.**
Définir `Sizer`, `Namer`, puis `SizedNamer` par composition. Écrire un type qui satisfait les
trois. Ajouter les vérifications `var _ SizedNamer = (*T)(nil)`. Que se passe-t-il si une seule
méthode manque ?

**E5 — Interface embarquée nil.**
Créer une struct embarquant une interface, **sans** l'initialiser, et appeler une méthode non
redéfinie. Relever la panique. Puis ajouter une garde. Dans quel contexte ce comportement
« panique si appelé » est-il en réalité **souhaitable** ?

---

## Exercice intermédiaire — `middleware`

Construire une chaîne de traitement décorée, sans HTTP — le patron pur.

```go
type Handler interface {
	Handle(req string) (string, error)
}
```

**À implémenter :**
- `EchoHandler` — retourne la requête telle quelle ;
- `UpperHandler` — décorateur qui met la réponse en majuscules ;
- `LoggingHandler` — décorateur qui journalise avant et après, avec la durée ;
- `RetryHandler` — décorateur qui réessaie jusqu'à N fois en cas d'erreur ;
- `ValidateHandler` — décorateur qui rejette les requêtes vides **sans** appeler le suivant ;
- `Chain(h Handler, decorators ...func(Handler) Handler) Handler`.

**Contraintes :**
1. Chaque décorateur **embarque l'interface `Handler`** et ne redéfinit que `Handle`.
2. Empiler quatre décorateurs et **prouver l'ordre d'exécution** par les logs. Prédire l'ordre avant de lancer.
3. `Chain` doit appliquer les décorateurs dans un ordre **documenté** : le premier de la liste est-il le plus externe ou le plus interne ? Trancher, écrire pourquoi, et rendre le comportement conforme.
4. `RetryHandler` ne doit pas réessayer une erreur de validation. *(Comment le décorateur peut-il distinguer une erreur « définitive » d'une erreur « temporaire » ? La réponse complète est à la leçon 6 — proposer une solution provisoire et noter ses limites.)*
5. Aucun décorateur ne connaît le type concret du décoré.
6. Ajouter un cinquième décorateur **sans modifier une seule ligne existante**. Le vérifier.

*Ce patron est exactement celui des middlewares `net/http` du niveau 7. Le construire ici,
sur un type trivial, permet de comprendre le mécanisme sans le bruit du HTTP.*

---

## Défi

**a) Faux partiel.**
Écrire une interface `Store` à six méthodes et un `fakeStore` de test qui n'en implémente
qu'une, en embarquant `Store`. Écrire un test manuel qui n'utilise que cette méthode, puis un
qui en appelle une autre — et observer.
Puis répondre : en quoi cette panique est-elle **préférable** à un faux qui retournerait des
zéro-valeurs pour les cinq autres méthodes ?

**b) Embedding et API publique.**
Créer `type SafeMap struct { sync.Mutex; data map[string]int }` avec des méthodes `Get` et
`Set` verrouillées. Montrer comment un code appelant extérieur peut provoquer un
**interblocage** en utilisant l'API publique — sans écrire une seule ligne incorrecte de son
point de vue. Corriger. Écrire trois lignes sur la leçon générale.

**c) Conception.**
Trois façons de réutiliser du comportement en Go : embedding, champ nommé + délégation
manuelle, fonction passée en paramètre. Prendre un cas concret — un système de notification
avec journalisation, réessai et limitation de débit — et le concevoir **trois fois**.
Comparer : lisibilité, testabilité, taille du code, facilité d'ajout d'un comportement,
risque d'API accidentelle. Y a-t-il un vainqueur, ou le choix dépend-il du cas ?

---

## Quiz

1. Qu'est-ce qu'un champ anonyme ? Quel nom porte-t-il implicitement ?
2. Que signifie « promotion » ?
3. Peut-on affecter un `Derived` à une variable de type `Base` ? Pourquoi ?
4. Si `Base.Describe()` appelle `Greet()` et que `Derived` redéfinit `Greet()`, quelle version est appelée ?
5. Que se passe-t-il si deux types embarqués ont un champ de même nom ?
6. Comment accède-t-on à une méthode masquée par le type extérieur ?
7. Qu'apporte l'embedding d'une **interface** dans une struct ?
8. Quel est le risque d'un champ embarqué valant `nil` ?
9. Pourquoi embarquer `sync.Mutex` publiquement est-il discutable ?
10. Quand préférer un champ nommé à un embedding ?
