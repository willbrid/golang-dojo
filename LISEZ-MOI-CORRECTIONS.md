# ⚠️ Branche `corrections`

Cette branche contient les **corrigés** des exercices, en plus de tout le contenu de `main`.

```
git checkout corrections   # consulter un corrigé
git checkout main          # revenir au cours
```

## La règle

Un corrigé lu avant d'avoir essayé n'apprend presque rien. Un corrigé **comparé à sa propre
tentative** apprend beaucoup : c'est l'écart entre les deux qui est instructif, pas la
solution elle-même.

Ordre recommandé pour chaque exercice :

1. Essayer, même mal, même incomplet.
2. Envoyer sa tentative au professeur et recevoir des indices.
3. Corriger soi-même.
4. **Ensuite seulement**, ouvrir le corrigé — pour comparer, pas pour découvrir.

## Ce que contient un `correction.md`

- le corrigé des exercices faciles, avec le point précis que chacun visait ;
- le corrigé complet et commenté de l'exercice intermédiaire ;
- la solution du défi, ou une piste détaillée quand plusieurs conceptions se valent ;
- les réponses argumentées du quiz.

Les corrigés expliquent surtout **pourquoi** une solution est préférable à une autre. Quand
plusieurs réponses se défendent, elles sont comparées plutôt que départagées arbitrairement.

## Maintenance

Cette branche suit `main`. Après une mise à jour du cours :

```bash
git checkout corrections
git merge main
```
