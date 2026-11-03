# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-12/lecon-03/`, en partant du service et du Dockerfile de l'exemple.

## Exercices faciles

**E1 — Réparer `scratch`.**
Modifier `Dockerfile.scratch` pour que le service y trouve les certificats racine (à copier depuis l'étape
de build) et les fuseaux horaires (`import _ "time/tzdata"`), et qu'il tourne sous l'uid 65532. Vérifier les
trois lignes du journal. Comparer la taille de l'image avec la version *distroless*. Que coûte `time/tzdata`
dans le binaire ?

**E2 — L'ordre des couches.**
Ajouter une dépendance au service (par exemple `golang.org/x/sync`, utilisée pour de vrai). Mesurer le temps
de reconstruction après la modification d'un fichier `.go`, avec le Dockerfile du cours, puis avec un
Dockerfile où `COPY . .` vient avant `go mod download` et sans caches montés. Que montre la sortie de
`docker build` (lignes `CACHED`) ?

**E3 — Ce que gardent les couches.**
Ajouter à l'étape de build `ARG TOKEN` et `RUN echo "jeton : $TOKEN" > /tmp/t && rm /tmp/t`, puis construire
avec `--build-arg TOKEN=s3cr3t`. Que montre `docker history --no-trunc` de l'image de l'étape de build
(`docker build --target build`) ? Et de l'image finale ? Comment passer un secret à un build sans qu'il reste
dans une couche (`RUN --mount=type=secret`) ?

**E4 — Arrêts lents.**
Ajouter une route `/lent` qui répond au bout de 15 s. Lancer une requête, puis `docker stop` pendant qu'elle
s'exécute. Que reçoit le client ? Combien de temps prend l'arrêt ? Que change `docker stop -t 30` ? Que faut-il
régler dans le service pour que son délai d'arrêt reste **inférieur** à celui de l'orchestrateur ?

**E5 — Réduire les droits.**
Lancer le service avec `--read-only --cap-drop=ALL --security-opt=no-new-privileges`. Fonctionne-t-il ?
Modifier son adresse en `:80` : que se passe-t-il sous l'utilisateur `nonroot`, puis en `root` avec
`--cap-drop=ALL` ? Et si le service écrit un fichier temporaire (`os.CreateTemp`) avec `--read-only` ?
Comment l'autoriser (`--tmpfs /tmp`) ?

---

## Exercice intermédiaire — Le service et sa base dans Docker Compose

Pour le service de prêts du niveau 10 (ou le projet 4) :

1. Écrire son Dockerfile selon le cours.
2. Écrire un `compose.yaml` avec deux services, `db` (`postgres:17`) et `api`. Pour `db`, une sonde de
   santé (`pg_isready`) et un volume nommé.

   Pour `api`, `depends_on` avec `condition: service_healthy`, la configuration par variables
   d'environnement (`DATABASE_URL`) et le port publié **sur 127.0.0.1 seulement**.
3. Les migrations (N8 L6) s'exécutent au démarrage du service, ou par une sous-commande `migrate`
   lancée comme un service Compose à part qui doit se terminer avant `api`. Choisir et justifier.
4. Vérifier :
   - `docker compose up` démarre les deux services dans l'ordre ;
   - `docker compose stop api` arrête l'API proprement (journal) ;
   - après `docker compose down`, puis `up`, les données sont toujours là.
5. Qu'est-ce qui ne doit **pas** figurer dans `compose.yaml` si on le valide dans le dépôt ?

---

## Défi — Une image pour deux architectures

1. Construire l'image pour `linux/amd64` et `linux/arm64` avec
   `docker buildx build --platform linux/amd64,linux/arm64`.

   Pour l'étape de build, écrire `FROM --platform=$BUILDPLATFORM golang:1.27.3 AS build` et utiliser les
   arguments automatiques `TARGETOS` et `TARGETARCH` : `GOOS=$TARGETOS GOARCH=$TARGETARCH go build …`.
   Pourquoi est-ce bien plus rapide que de laisser l'étape de build s'exécuter dans un ARM émulé ?
2. Si l'émulation (QEMU, `binfmt`) est disponible sur ta machine, mesurer le temps des deux variantes
   (avec et sans `--platform=$BUILDPLATFORM`). Sinon, expliquer pourquoi la seconde variante échouerait.
3. Vérifier que l'image contient bien deux variantes (en l'exportant au format OCI, par exemple), et que le
   binaire de chacune est de la bonne architecture.
4. Les binaires des deux variantes sont-ils reproductibles d'un build à l'autre ? Et les images
   elles-mêmes (leurs empreintes) ? Qu'est-ce qui change dans une image entre deux builds identiques ?

---

## Quiz

1. Qu'est-ce qu'un Dockerfile multi-étapes, et pourquoi l'utiliser pour Go ?
2. Que manque-t-il dans `scratch` pour un service qui fait du HTTPS et manipule des fuseaux horaires ?
3. Qu'apporte l'image *distroless* `static:nonroot` ?
4. Dans quel ordre copier les fichiers pour profiter du cache ?
5. À quoi servent les caches montés (`--mount=type=cache`) ?
6. Pourquoi exclure `.git` du contexte, et quelle conséquence pour la version ?
7. Que fait `docker stop`, et pourquoi un shell en processus 1 le fait-il attendre 10 secondes ?
8. Pourquoi la forme tableau de `ENTRYPOINT` ?
9. Comment écrire une sonde de santé dans une image sans shell ni `curl` ?
10. Pourquoi ne jamais passer un secret en `ARG` ?
