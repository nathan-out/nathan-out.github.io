---
title: "So you hacked a wifi router and now what ?"
date: 2022-06-30T13:56:22+02:00
draft: false
---
*Damien Cauquil*

Le conférencier veut nous inviter à aller plus loin que l'accès root. Il prend l'exemple d'un modèle de routeur où une faille 0-day a été découverte et qui ne sera pas patchée par le fabricant. Qu'est-ce qu'il est alors possible de faire ? Est-il possible d'avoir un accès **total** à la machine ? Pas seulement root, mais vraiment total ; pouvoir installer un autre firmware dessus, pose de backdoor, pivot...

Le système est un linux embarqué. Son boot process est le suivant : boot ROM -> Bootloader -> Kernel. Le Secure Boot est un méchanisme de protection qui permet de s'assurer que chaque composant du processus de boot est sain. Si un tel méchanisme est en place sur cet appareil, il faudra le neutraliser ; sinon l'appareil ne bootera pas.

L'appareil comporte également une mémoire type NAND flash. C'est une mémoire peu fiable qui possède un méchanisme de nivellement d'usure et un mécanisme de code correcteur d'erreur (ECC).

## Problèmes post-exploit

Grâce à l'exploit de la 0-day nous sommes root, et ensuite ? On pourrait créer un botnet, prendre le contrôle de l'entiereté du système ou bien utiliser la machine pour du rebond.

**On souhaite garder un accès à la machine et bloquer les éventuelles mises-à-jour.**

Problème : la mémoire du système est une ROM (read only memory), comment laisser une backdoor sur un système en lecture seule ? On pourrait alors dumper la mémoire, l'éditer puis la flasher de nouveau sur la ROM. Cependant, c'est une opération risquée en soit, et encore plus si la machine possède un Secure Boot. Pour vérifier ce dernier point, il faut fouiller dans le firmware de l'appareil. En faisant du reverse, on s'apperçoit qu'il y a des conditions étranges au tout début du démarrage de la machine, c'est un **Secure Boot maison**. Il possède une vulnérabilité : il ne vérifie pas l'intégralité du Bootloader, en plus du fait que les vérifications ne semblent être que deux conditions. Il suffit de les mettre à vrai pour outrepasser le Secure Boot.

Une fois cette étape terminée, il n'y a plus aucune sécurité. On peut alors installer [OpenWRT](https://openwrt.org/) (distribution Linux pour du matériel embarqué).

**Conclusion**

- Un shell root ne veut pas dire qu'on a compromis tout le système.
- Le Secure Boot n'est pas infaillible, surtout lorsque c'est une solution maison.
- La programmation des mémoires types NAND flash est quelque chose de risqué.
