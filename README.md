# nathan-out.github.io

Mon blog GoHugo. La branche `gh-pages` contient les sources du site, la branche `master` tout le projet GoHugo.

## Ajouter un article

`hugo new posts/my-first-post.md`

## Build le site

`hugo`

## Mettre à jour le site

- build le site `hugo`
- ajouter les nouveaux fichiers `git add .`
- commit & push

```
$ hugo # génère les pages html
$ git add .
$ git commit -m "message de commit"
$ git checkout master
$ git push
$ git checkout gh-pages
$ git pull
$ git add .
$ git commit -m ""
$ git push
```

## Modifier le style

Ma feuille de style personnalisée : `assets/css/custom.css`
doc : https://jpanther.github.io/congo/docs/advanced-customisation/#overriding-the-stylesheet
