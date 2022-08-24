---
title: "Honeypot in ICS environment"
date: 2022-06-30T13:56:22+02:00
draft: false
---
*CyberSec ICS*

<u>**Honeypot</u>** :
- service volontairement exposé sur le net pour faire de l'analyse de le menace.
- 3 types : low, medium et high interaction. Pour faire simple, c'est le niveau de crédibilité du honeypot.

<u>**Objectif</u>** : créer un honeypot crédible pour des systèmes industriels et analyser les données obtenues.

## 1. Rechercher des équipements industriels exposés sur internet

Utilisation de [Shodan](https://www.shodan.io/). Beaucoup de machines exposées, plus ou moins critiques, les données récoltées serviront à créer des honeypots crédibles.

## 2. Créer un honeypot crédible

Utilisation de python, il existe des librairies existantes. Cependant, certaines ont besoin d'être retravaillées, en plus de certaines interfaces réseau qu'il faut recoder. Il faut aussi que le honeypot dispose d'une intéraction, qu'il soit "high interaction". Cette partie est primodiale pour que la collecte de données soit représentative de l'état de la menace.

## 3. Hébergement

Au départ il était question d'héberger les honeypots dans le monde entier. Cependant, pour rester crédible il ne faut pas héberger ce type de système sur des hébergeurs type AWS ou Microsoft ; ils ne sont traditionnellement pas dans l'hébergement de systèmes industriels. Ce sont plutôt des petits hébergeurs locaux et dont les prix sont plus élevés. Quelques pays ont été privilégiés, dont l'Ukraine.

## 4. Résultats

[Shadowserver](https://www.shadowserver.org/) : scanneur très utile pour les systèmes industriels.

- Beaucoup de passages de scanners (beaucoup aux USA, un peu Pays-Bas et Islande [deux pays qui possèdent beaucoup de datacenters]).
- D'autres visite (hors scans) de la part des USA, Chine, Brésil, Chine et Russie.
- Tous les protocoles ne sont pas visés équitablement.
- Les attaquants reviennent généralement sur MOD-BUS, FOX et OPCUA.
- Les mot-de-passe utilisés en bruteforce dépendent fortement de la zone géographique dans laquelle est situé le honeypot.

## Conclusion

- Il est possible de monitorer et de détecter des nouveaux comportements de la part des acteurs malveillants.
- Il n'est pas toujours facile de déployer des honeypots crédibles (par exemple le conférencier à dû modéliser mathématiquement une usine de traitement de l'eau et recréer une interface graphique).

Pour aller plus loin, il faudrait installer des honeypots directement dans des entreprises. De plus, il faudrait travailler sur la couche TCP/IP (vulnérabilite [amnesia33](https://www.forescout.com/research-labs/amnesia33/)).
