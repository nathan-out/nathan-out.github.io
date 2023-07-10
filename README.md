# nathan-out.github.io

Mon blog GoHugo.

**Note : problème avec la dernière versio, impossible de build le site. Version fonctionnelle : 0.110.0.**

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
