# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-05/lecon-03/`.

## Exercices faciles

**E1 — Externe.**
Passer le test de l'exemple en `package reminder_test`. Qu'est-ce qui doit changer ? Qu'est-ce
que ça prouve sur le couplage entre `reminder` et son `mailer` ?

**E2 — Trouver les dépendances cachées.**
Lister **toutes** les dépendances cachées de cette fonction, puis la réécrire pour qu'elle soit
testable sans toucher au disque ni à l'horloge ni à l'environnement :

```go
func Backup() error {
	dir := os.Getenv("BACKUP_DIR")
	name := fmt.Sprintf("backup-%s.tar", time.Now().Format("20060102"))
	f, err := os.Create(filepath.Join(dir, name))
	if err != nil {
		return err
	}
	defer f.Close()
	log.Printf("sauvegarde dans %s", name)
	return writeArchive(f, globalDB)
}
```

**E3 — Stub, fake, spy, mock.**
Pour chacun des doubles suivants, dire s'il s'agit d'un stub, d'un fake, d'un spy ou d'un mock :
(a) un `Clock` qui rend toujours le même instant ; (b) une `map` qui implémente un `UserStore`
avec `Get`, `Put`, `Delete` et `ErrNotFound` ; (c) un `Notifier` qui ajoute chaque message à un
slice ; (d) un objet généré qui échoue si `Close` n'a pas été appelé après `Open`.

**E4 — L'erreur que le fake ne simule pas.**
Écrire un fake de `UserStore` qui ne rend jamais `ErrNotFound`. Écrire un service qui oublie de
gérer ce cas. Montrer que le test passe. Corriger le fake, et montrer que le test révèle le bug.

**E5 — Pas de `Sleep`.**
Voici un test :

```go
func TestCacheExpires(t *testing.T) {
	c := NewCache(100 * time.Millisecond)
	c.Set("k", "v")
	time.Sleep(150 * time.Millisecond)
	if _, ok := c.Get("k"); ok {
		t.Error("la clé aurait dû expirer")
	}
}
```

Pourquoi est-il mauvais (deux raisons) ? Modifier `Cache` pour qu'il reçoive son horloge, et
réécrire le test pour qu'il s'exécute en moins d'une milliseconde.

---

## Exercice intermédiaire — Le service de paiement

```go
type PaymentGateway interface { … }   // à définir — et c'est la moitié de l'exercice

type Checkout struct { … }

// Pay débite la carte, enregistre la commande, envoie un reçu.
func (c *Checkout) Pay(order Order, card Card) (Receipt, error)
```

Règles métier :
1. Le débit passe par une passerelle externe (lente, payante, non déterministe).
2. Si le débit réussit mais que l'enregistrement de la commande échoue, il faut **rembourser**.
3. L'envoi du reçu peut échouer sans annuler le paiement (on journalise, on rend quand même le reçu).
4. Une même commande ne doit jamais être débitée deux fois (clé d'idempotence = identifiant de commande).

**À faire :**
- Définir les **interfaces côté consommateur** : combien, avec quelles méthodes ? Justifier chaque
  méthode : est-elle vraiment utilisée par `Checkout` ?
- Écrire des fakes pour chacune, capables de simuler les pannes.
- Écrire une table de tests couvrant : succès, débit refusé, débit OK + enregistrement KO
  (→ remboursement vérifié), reçu KO, double paiement.
- **Aucun** test ne vérifie l'ordre des appels ; tous vérifient l'**état final** des fakes.
- Question : le fake de passerelle doit-il implémenter l'idempotence lui-même ? Que testerait-on
  sinon ?

---

## Défi — Le mock qui casse au refactoring

1. Avec `go.uber.org/mock` (`mockgen`), générer un mock de `mailer` et réécrire `TestRemind` avec
   des attentes (`EXPECT().Send(…).Times(1)`).
2. Refactoriser `Remind` pour envoyer **un seul** courriel récapitulatif par destinataire au lieu
   d'un par tâche. Lancer les deux versions du test (fake et mock). Laquelle casse ? Laquelle
   **aurait dû** casser ?
3. Trouver un cas où le mock détecte un vrai bug que le fake ne voit pas. *(Indice : un appel
   qui ne doit **jamais** avoir lieu.)* Comment le fake pourrait-il le détecter aussi ?
4. Conclure en cinq lignes : dans quelles circonstances un mock est-il le bon outil ?

---

## Quiz

1. Qu'est-ce qu'une dépendance cachée ? Donner trois exemples.
2. Quelle est la première façon d'injecter une dépendance à envisager ?
3. Pourquoi définir l'interface du côté du consommateur ?
4. Différence entre un fake et un mock ?
5. Pourquoi un test à base de mocks casse-t-il au refactoring ?
6. Que doit savoir simuler un bon fake, en plus du cas nominal ?
7. Quelles dépendances ne faut-il pas doubler ?
8. Pourquoi `time.Sleep` dans un test est-il une mauvaise idée ?
9. Pourquoi remplacer une variable globale dans un test pose-t-il problème ?
10. Quel paquet Go 1.25 permet de tester du code concurrent avec une horloge virtuelle ?
