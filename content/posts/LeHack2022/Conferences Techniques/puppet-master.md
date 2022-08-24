---
title: "How to become the socket puppets master ?"
date: 2022-06-30T13:56:22+02:00
draft: false
---
*Palenath Megadose*

La conférence parle essentiellement de la création gratuite et facile de numéros de téléphone, utiles dans le cadre de la création de faux comptes de réseaux sociaux. Le conférencier veut pouvoir créer et manager facilement des numéros de téléphones qui ne pourraient pas lui être imputés.

Le présentateur nous indique plusieurs solutions :
- il existe des cartes SIM gratuites à disposition dans certains magasins de téléphonie/bureaux de tabac. 
- les différents services en lignes ne sont bien souvent pas gratuits, leurs numéros sont connus de la plupart des réseaux sociaux et il est difficile de passer à l'échelle.
- Arduino : intéressant d'un point de vue industrialisation mais bien trop cher, il faudrait un arduino par carte SIM.
- Dongle 4G -> cher
- Sim box -> intéressant, le coût n'est pas très élevé mais il y a de gros inconvénients. Les logiciels sont difficilement accessibles sous Linux, ils sont chers pas toujours très puissants.

Il a donc développé lui-même son logiciel de gestion. Le conférencier nous indique qu'il utilise des commandes AT (commandes pour des modems), cela permet d'intéragir avec les SIM. Il supprime systématiquement les SMS des SIM pour rester anonyme. La gestion des cartes SIM est grandement facilitée.

Pour automatiser la création de compte sur les réseaux sociaux :
- créer son proxy 4G
- utilisation de Selenium undetectable pour outrepasser les protections anti-bots
- utilisation de [Buster](https://github.com/dessant/buster) (résolution de captcha)
