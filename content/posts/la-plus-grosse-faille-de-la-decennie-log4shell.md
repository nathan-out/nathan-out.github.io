---
title: "La Plus Grosse Faille De La Décennie Log4shell"
date: 2021-12-17T13:49:02+02:00
draft: false
---
*Je vais vous expliquer simplement comment cette faille, découverte le 10 décembre 2021, est en mesure de mettre à feu et à sang l’écrasante majorité des systèmes informatiques. La légende voudrait qu’elle ait été découverte sur un serveur Minecraft dont l’unique but était de faire des farces. Cette faille a le bon goût de ne pas être trop technique, laissez vous guider, je vous explique tout …*

![Une illustration qui représente bien l'état d'esprit de certains responsables de la cybersécurité en ce moment...](/img/blog/log4j-faille-fire-elmo.jpg)

## D'abord une librairie : log4j

Tout commence par la librairie **log4j** qui est écrite en **Java** (oui le même truc qui vous demande souvent une mise à jour). Une librairie c’est du **code écrit par des développeurs pour automatiser des tâches et qui est ensuite réutilisée par d’autres développeurs dans des projets plus gros**. C’est un peu comme une brique spécifique qui permet de construire plus rapidement des maisons. Une grande partie des applications, des sites, des logiciels utilisées sur le web (ou reliés au web) utilise log4j et donc Java. Cette librairie sert essentiellement à gérer les *logs*, c'est-à-dire la trace de ce qu'il se passe sur le serveur.

## Ensuite la faille : log4shell

Log4shell est le nom de la faille, et elle se base sur une vulnérabilité de log4j. Mais comment ça fonctionne ? En fait log4j pour fonctionner reçoit des données et les traite. Cependant, dans les données qu’on lui envoie on peut lui signifier des tâches automatiques à faire (par exemple, ajouter dynamiquement l'heure), et notamment lui demander d’aller chercher du contenu quelque part sur internet. Si on demande à log4j d’aller chercher du code Java… **la librairie va exécuter ce code !** C’est ça la faille log4shell.

Cela signifie que si j’envoie la bonne donnée à un serveur, celui-ci demande à log4j de la traiter. Cependant cette donnée dit à la librairie d’aller chercher une ressource sur internet qui n’est autre que du code Java malveillant que j’ai moi-même codé. Une fois ce code récupéré, le serveur va **l’exécuter**. Je peux alors exécuter n’importe quel code Java sur un serveur ne m’appartenant pas. On peut utiliser ce code pour créer un ***« reverse shell »*** (d’où le nom log4shell) : un accès direct, non autorisé et avec tous les droits sur le serveur. On a donc un accès total et absolu sur la machine ciblée.

*(en vérité le niveau d'accès dépend de plusieurs facteurs, mais souvent on peut obtenir des accès assez importants voire carrément administrateurs)*

On peut ensuite utiliser cet accès pour essayer d’accéder à d’autres machines, créer un autre accès plus discret, mettre hors service le serveur, voler des données…

## Log4j everywhere

Cette librairie est **partout**, et sa faille avec. Un exemple : **le rover de la NASA actuellement sur Mars utilise log4j**. Les pirates peuvent donc librement scanner internet à la recherche d’un serveur, d’une application, d’un site qui utilise cette librairie pour ensuite prendre le contrôle de la machine. C’est d’ailleurs ce qu’ils font depuis que la faille a été exposée au grand jour, comme on peut le voir sur ce site qui répertorie les tentatives d’exploitation de la vulnérabilité : [log4j-tracker](https://crowdsec.net/log4j-tracker/). Mais il y a pire, encore faut-il savoir qu'on utilise cette librairie ! Les logiciels sont de plus en plus complexes et embarquent avec eux des librairies... qui embarquent elles-mêmes des librairies, et ainsi de suite ! On peut donc être vulnérable au second, troisième... n-ième degré, sans le savoir.

A l’heure de l’écriture de ces lignes (17 décembre 2021) c’est une course contre les pirates qui est menée par les responsables cybersécurité de toutes les entreprises/organismes qui utilisent cette librairie. Certains publient des correctifs alors que dans le même temps des pirates cherchent à les contourner.

Mes pensées vont à tous les responsables cyber qui passeront (à minima) des fêtes de fin d’année mouvementées…
