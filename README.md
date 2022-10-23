# nathan-out.github.io

Mon blog GoHugo. La branche `gh-pages` contient les sources du site, la branche `master` tout le projet GoHugo.

## Ajouter un article

`hugo new posts/my-first-post.md`

## Créer les pages HTML

`hugo`

## Mettre à jour le site

Il faut push sur les deux branches :
- `main` qui contient les sources
- `gh-pages` qui contient les html générés

```
$ git add .
$ git checkout main
$ git push
$ git checkout gh-pages
$ git push
```
