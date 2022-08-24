---
title: "OSINT et les investigations judiciaires"
date: 2022-06-30T13:56:10+02:00
draft: false
---

*Lieutenant Yann Derweduwen (C3N)*

Le ComCyberGend communique beaucoup avec d'autres organismes nationaux comme l'ANSSI les assureurs ou encore la Police Judiciaire. L'Europe n'est pas en reste non plus puisqu'une communication privilégiée avec Interpol est également en place. 

L'OSINT se situe dans ce processus de travail :

Accueil du public -> OSINT -> ESP* basse intensité -> ESP (infiltration active, filature, planque ...)

*ESP : enquête sous pseudonymat (infiltration). L'ESP de basse intensité ne comporte pas de contact direct avec d'éventuels malfaiteurs.


L'OSINT au sens du gendarme ne comporte un travail que sur ce qui pourra être jugé ensuite. Il n'y a pas de collecte des opinions politiques, d'une tendance d'un individu, si cela ne permet pas de constituer une preuve. C'est la différence entre le **renseignement** -> anticiper et le **judiciaire** -> punir. A la différence du renseignement, le judiciaire ne fait que du **légal** et la façon d'obtenir des informations est importante. En France la preuve est libre, on travail à charge **et** à décharge, cela n'est pas le cas dans tous les pays.

D'une manière générale, on applique les mêmes techniques dans le cyberespace que dans le "monde réel". Le cyber est un espace comme les autres, au même type qu'un milieux montagnard : il a des règles et des limitations intrinsèques.

La première chose à faire est d'étudier l'environnement immédiat, on va ensuite en élargissant : 
- gel des lieux -> on fige les preuves (scrapping)
- planque & filature -> surveillance des réseaux sociaux, d'éventuel site sur les darknets ...

<u>Il est compliqué d'utiliser une détection par les signaux faibles. Cela demande de très gros moyens et ça n'est pas toujours légal. L'exploitation de fuite de base de données est considéré comme du recel.</u>

<br><br>

**<u>GEOTROUVETOUT</u>**

[Overpass Turbo](https://overpass-turbo.eu/) qui était utilisé avant par le C3N, possède plusieurs limitations :
- langage de requête complexe
- documentation perfectible
- pas de sauvegarde

C'est pourquoi le C3N a développé un outil qui se base sur Overpass Turbo, baptisé Geotrouvetout. Il suffit de décrire un lieux (présence d'un passage piéton, d'une pharmacie à moins de 50m, un feux tricolore ...) et l'outil sort des lieux probables.

Un exemple d'utilisation de l'outil a été exposé : un vrai appel au SDIS31 avec un interlocuteur très confus. La personne est en état de choc, son ami vient de se brûler le visage et il ne semble pas en mesure de décrire l'endroit où il se trouve, il n'est pas précis. Le pompier lui demande de décrire le lieux autour de lui et grâce à ces éléments et quelques déductions, il est possible de trouver le lieux de l'appel et d'envoyer une équipe très rapidement grâce à l'outil.
