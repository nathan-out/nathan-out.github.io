---
title: "Stuxnet : Infiltrer Et Compromettre Une Centrale Nucleaire"
date: 2021-11-21T10:51:19+02:00
draft: false
---

Comment les centrales nucléaires iraniennes on-t-elles été infiltrées et sabotées par un programme informatique ?

*Cet article est un issu d'un devoir d'analyse de la menace effectué durant ma première année d'école d'ingénieur.*

## Tout commence en Biélorussie...

En juin 2010, la société biélorusse VirusBlockAda reçoit un appel d’un de ses clients irannien. Certains ordinateurs ont un comportement étrange, ils redémarrent sans raison et indiquent des BSOD (erreur système fatale). Ce qui semblait être une infection mineure, s’avère être bien plus mystérieux quand l’équipe biélorusse découvre qu’un programme malveillant est à l’œuvre et que celui-ci est **signé par des sociétés comme Verisign ou encore Realtek Semiconductor Corps**, des compagnies reconnues, sérieuses et fiables.

### Quelques mots sur la signature

La signature est un procédé cryptographique qui permet **d'approuver la provenance et l'intégrité** (non-modification) de l'élément signé, ici un programme. Cela permet d'être certain que le programme provient bien d'une organisation de confiance. C'est cette organisation qui signe le logiciel grâce à un **certificat**, et chacun peut ensuite vérifier que la signature est bien celle de l'organisme émetteur.

Les certificats sont créés par l'organisme et sont **extrêmement bien protégés**. En effet, si un pirate pouvait obtenir un certificat d'une société comme Google, il pourrait créé un programme se revendiquant de cette entreprise. Les machines approuvant les programmes de chez Google laisseraient donc aussi ce programme passer.

## La première cyber-arme de l'Histoire

Après des recherches plus poussées, il s’avère que ce programme utilise quatre *« zero days » (failles pour lesquelles il n'existe pas de correctif, souvent inconnues avant leur exploitation)* et qu’il passe outre la plupart des antivirus. VirusBlockAda venait de découvrir **Stuxnet**, la première *« cyber arme »* connue de l’histoire qui visait des systèmes industriels très précis, à savoir des **centrifugeuses iraniennes d’enrichissement d’uranium**. Ces sites hautement sensibles et protégées se sont soudainement rendues compte que leurs machines étaient infectées depuis au moins un an par un ***« APT »*** *(advanced persistant threat = menace avancée persistante)*. Pire que cela, après des mois d’investigation, les iraniens ont reconnu que Stuxnet avait réussi à *« causer des problèmes à un nombre limité de nos centrifugeuses »* mais aussi que le virus aurait été introduit via des **clefs usb infectées**. Enfin, selon Microsoft, **une équipe sur place aurait été nécessaire** afin d’avoir une connaissance de l’environnement suffisante pour développer le programme. Plusieurs questions se posent :
- comment une attaque d’une telle complexité, d’une telle envergure, a-t-elle pu être menée à bien ?
- comment est-ce possible qu’un virus puisse impacter ce type d’installation ?
- pourquoi est-il passé sous les radars ?

## Contexte

Cette attaque inédite s’inscrit dans un environnement géopolitique complexe entre l’Iran, ses voisins (notamment Israël), et les puissances occidentales. Depuis des décennies, l’Iran mène un **programme nucléaire civil**. Cependant, plusieurs puissances étrangères pensent que ce programme de recherche cache un côté militaire visant à **l’acquisition de l’arme nucléaire**. Cela menace l’équilibre géopolitique précaire de la région, en particulier avec Israël. Ce programme, quel que soit son objectif, nécessite des **centrifugeuses** et c’est l’entreprise Siemens qui les fournit. C’est un des points clefs du projet et c’est à ces dispositifs que Stuxnet s’attaque. Aujourd’hui, il semblerait que cette attaque ait été orchestrée par les États-Unis dans le cadre de l’opération ***« Olympic Games »*** (voir l'affaire Snowden), avec l’aide d’Israël. Ce dernier ne se cachant plus que la cyberguerre fait désormais partie intégrante de sa doctrine militaire.

A noter que ni les Etats-Unis, ni Israël n'ont revendiqués l'attaque. Il y a toujours des doutes quant aux commanditaires de cette opération. Plusieurs éléments semblent indiquer qu'il pourrait s'agir de ces deux pays. De manière générale, un état ne revendique jamais ses actions offensives dans le cyber-espace (guerre en Ukraine à part).

## Exploitation

Ce virus se propage de trois manières différentes, via des clefs USB, des fichiers projets Siemens vérolés, ou par réplication dans le réseau. De plus, il est capable **d’établir un réseau peer-to-peer pour se mettre à jour même sans connexion internet**. En se répliquant, le virus est capable de communiquer avec ses copies, si l'une de ces copies possède un accès à internet, elle va se mettre à jour et transmettre les mises à jour aux autres copies sur le réseau (bien souvent non connnectées à internet).

Stuxnet dispose de deux atouts pour rester discret : il est signé par des entreprises officielles et **il peut modifier son comportement en fonction des antivirus installés sur les machines**. A noter que ce programme vise des installations très spécifiques. S’il ne se trouve pas sur l’une d’elles, il se réplique sur les machines à sa portée et se supprime. Il est important de comprendre la subtilité avec laquelle Stuxnet vise ses cibles. Ce programme a été retrouvé a posteriori dans plusieurs systèmes industriels de par le monde, notamment dans une **centrale nucléaire russe**. Cela grâce à l’efficacité de la propagation du vers (type de virus). Cependant, les cibles semblent être les centrales nucléaires iraniennes, aux vues des versions et configurations extrêmement précises requises pour que Stuxnet opère. Cela renforce également les soupçons **d’une aide interne** pour obtenir ces informations ultra-sensibles.

Vient ensuite la phase d’installation et de persistance durant laquelle il utilise une faille zero-day pour élever ses privilèges et se copier dans des processus systèmes de Windows, le rendant quasiment invisible. Une fois arrivé à son objectif, Stuxnet communique avec l’API de la centrifugeuse et **modifie les données envoyées et reçues**. Se faisant, il **masque les alertes de sécurité et change périodiquement la vitesse de rotation des centrifugeuses**. L'objectif visé était **d'endommager les centrifugeuses** pour retarder le programme nucléaire irannien. Dans le même temps, le système de pilotage indique un comportement normal de l’automate. On retrouve ici les caractéristiques types d’une APT avec **beaucoup d’efforts investis sur la persistance de Stuxnet**.

Enfin, l’objectif de Stuxnet était vraisemblablement de ne **jamais être découvert** puisqu’une date d’effacement du programme était fixée au 24 juin 2012. Ce programme étant décrit comme ***« la cyber arme la plus complexe de l’Histoire »*** dispose même d’un processus d’effacement complexe et évolué. Ce code a été ensuite analysé puis récupéré par des agences étatiques et des cyber-criminels pour produire d’autres virus redoutables tels que **Flame** ou **Duqu**. Flame serait à l’origine d’attaques visant la destruction de données.

## Ce qu'il faut retenir

Cette opération nous montre qu’une attaque d’une telle envergure n’est possible qu’avec **le travail d’une ou plusieurs puissances étrangères aux moyens et capacités opérationnelles importants**. Elle nécessite de disposer de **renseignements très précis** et fiables qui ne peuvent être dispensés que par ceux travaillant **au plus proche des cibles**. On observe également qu’une avance technologique est obligatoire ainsi que des **exactions illégales préalables** (certificats volés, espionnage). La détection de pareilles menaces est difficile et leurs exactions est parfois **invisible**. 

**La cyberguerre est largement mondialisée et il convient de s’y préparer, autant offensivement que défensivement.**

## Source

Pour aller plus loin je vous recommande vivement les articles ci-dessous et pour les plus techniques cette conférence Youtube par un employé de Microsoft qui explique le fonctionnement de Stuxnet : [27C3: Adventures in analyzing Stuxnet (Bruce Dang from Microsoft)](https://www.youtube.com/watch?v=rOwMW6agpTI).

- [The man who found Stuxnet, Sergey Ulasen in the spotlight - Eugene Kaspersky](https://eugene.kaspersky.com/2011/11/02/the-man-who-found-stuxnet-sergey-ulasen-in-the-spotlight/)
- [Stuxnet - Wikipedia](https://en.wikipedia.org/wiki/Stuxnet)
- [Stuxnet - Generation NT](https://www.generation-nt.com/stuxnet-ver-informatique-iran-nucleaire-actualite-1122561.html)
- [Programme nucléaire irannien - Wikipedia](https://fr.wikipedia.org/wiki/Programme_nucl%C3%A9aire_de_l%27Iran)
- [Virus Stuxnet, l'arroseur arrosé - Infoguerre](https://www.ege.fr/infoguerre/2012/11/virus-stuxnet-l%25e2%2580%2599arroseur-arrose)
- [Analysis shows traces wiper malware no links flame - Threatpost](https://threatpost.com/analysis-shows-traces-wiper-malware-no-links-flame-082912/76960/)
- [Opération Olympic Games - Wikipedia](https://en.wikipedia.org/wiki/Operation_Olympic_Games)
- [Signature de code - Wikipedia](https://fr.wikipedia.org/wiki/Signature_de_code)
