---
title: "Espace vide & superblock : trifouiller EXT4 pour le fun (2/2)"
date: 2024-07-21T18:27:27+02:00
draft: false
---

## Résumé

Cet article est la seconde partie de [Espace vide & superblock : trifouiller EXT4 pour le fun (1/2)](https://nathan-out.github.io/posts/espace-vide--superblock-trifouiller-ext4-pour-le-fun-1-2/). Il détaille la résolution du challenge en plus d'apporter quelques éléments de compréhension des systèmes de fichiers et en particulier `ext4`. Ce challenge comporte deux parties : la première qui consiste à monter et déchiffrer une partition `LUKS`, la seconde à trouver un fichier caché dans l'espace vide (*slack space*).

Si vous êtes intéressé par la dissimulation d'information dans une partition, je vous invite à directement sauter à la partie 2. Gardez à l'esprit qu'il s'agit d'une introduction et qu'il existe bien d'autres techniques pour cacher de la donnée dans une partition. L'outil [Fishy](https://github.com/dasec/fishy), qui a été utilisé pour créé le challenge, est riche d'information à ce propos.

## Partie 1

Difficulté : facile

> Lors d'une précédente mission, nos forces ont trouvé une clef USB portant l'inscription "milou13041982". Vraissemblablement, nous n'aurions pas dû trouver cette clef. Elle pourrait contenir des informations d'intérêt pour le renseignement. Nos analystes ont fait une image de la seule partition contenue sur la clef USB, à vous de la faire parler.

Hints :

1. Si vous êtes perdu : que donne la commande `file` sur ce fichier ? -10pts

### Étape préliminaire

La première étape est de savoir à quoi on a à faire exactement :

```bash
file dump.dd 
dump.dd: LUKS encrypted file, ver 2 [, , sha256] UUID: 7c7cb115-5dbd-4019-affa-79ea508c1dd0
```

C'est donc une image d'une partition chiffrée avec [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup), une spécification de chiffrement de support de stockage. Fort heureusement, on donne laisse sous-entendre le mot de passe de la partition : `milou13041982`.

### Déchiffrement

Pour monter un disque, il faut basiquement créer un point de montage et monter la partition dessus. Cependant, puisque la partition est chiffrée il va falloir passer par quelques étapes en plus. On commence par déchiffrer la partition.

```bash
# association de l'image à un périphérique de boucle
$ sudo losetup -f --show dump.dd
/dev/loop2
$ sudo cryptsetup luksOpen /dev/loop2 chall
# on rentrera ici le mot de passe de déchiffrement
```

### Monter

Pour la monter, il faut créer un point de montage et monter la partition déchiffrée dessus : 

```bash
# création du point de montage
$ sudo mkdir /mnt/chall_mnt
# montage
$ sudo mount /dev/mapper/chall /mnt/chall_mnt
# on donne les droits pour plus de praticité
$ sudo chmod -R 777 /mnt/chall_mnt
$ ls /mnt/chall_mnt
lost+found  milou_v3.jpeg  operation_leviathan.pdf  tintin_fanart.jpg
```

Il ne reste qu'à se rendre dans `/mnt/chall_mnt` et le flag se trouve dans `operation_leviathan.pdf`: `FLAG{juste_une_partition_chiffree_avec_luks}`.

## Partie 2

Difficulté : difficile

> Les analystes du renseignement sont dubitatifs... ils ne semblent pas convaincus par ce soit-disant fichier top-secret. Certains pensent qu'il s'agit d'un fichier authentique, alors que d'autres conjecturent que ce n'est rien d'autre qu'un leurre destiné à nous tromper ! Ils décident de vous demander une analyse supplémentaire pour déceler un éventuel piège. Des données pourraient être cachées dans la partition **déchiffrée**.

Indication supplémentaire : les indices sont là pour vous aider. Si vous bloquez, n'hésitez pas à les prendre ; ils sont conçus pour vous aider pas à pas à résoudre le challenge. Résoudre le challenge sans aucun indice semble vraiment difficile (avis de l'auteur).

0. Il faut d'abord faire un dump (image) de la partition déchiffrée, un outil pratique pour ce genre de chose est `dd`. (gratuit)

1. Vous êtes à la recherche d'une archive zip. Ne faites pas confiance au système de fichier : il pourrait vous *cacher* des choses. -10pts

2. Le `magic number` de l'archive a été **légèrement** obfusqué pour ne pas que le challenge soit résolvable avec un simple outil de carving type `scalpel` ou `foremost` ;). -10pts (gratuit)

3. Connaissez-vous les *superblock* en `ext4` ? -10pts

4. `dumpe2fs` donne pas mal d'informations sur la partition, comme la taille des *blocks*. Avec ça, vous devriez savoir où chercher :) -10pts

5. Toujours pas trouvé ? Mais enfin regardez ce qu'il se cache dans le *superblock* ! N'y a-t-il pas quelque chose d'étrange ? N'ayez pas peur d'utiliser un éditeur hexa type `GHex`. -10pts

6. Il va faloir mettre les mains dans le cambouis... le fichier caché fait 610 octets tout pile -10pts

7. Les blocks de cette partition font 4096 octets (`dumpe2fs test_zip.dd|grep size`). Le `superblock` commence à l'offset 0. Donc il faut chercher dans les 4096 premiers octets, on y remarque la présence d'un fichier à l'offset 2048 (magic number `QL` qui est en fait le magic number obfusqué d'une archive zip `PK` +1). On sait que le fichier fait 610 octets ! On n'a plus qu'à le dump et modifier le magic number : `dd if=image.dd bs=1 skip=2048 count=610 > archive.zip` -20pts

### Préparation

On nous dit qu'il faut chercher sur la partition **déchiffrée**. On va donc effectuer une image de cette partition : si des données sont cachées, il va falloir aller regarder dans le détail.

J'admets que la partition est déjà déchiffrée et montée (partie 1 du challenge).

```bash
$ sudo dd bs=1 if=/dev/mapper/partition of=partition_dechiffree.dd
```

Si ce n'est pas le cas, voici les commandes pour monter le dump chiffré, le déchiffrer, et faire votre propre dump :

```bash
# cette commande va vous donner un numéro de loop, elle sera utile ensuite
sudo losetup -f --show dump.dd
# déchiffrement
sudo cryptsetup luksOpen /dev/loop12 partition
# montage
sudo mount /dev/mapper/partition /mnt
# on a bien les fichiers
ls /mnt
lost+found  milou_v3.jpeg  operation_leviathan.pdf  tintin_fanart.jpg
# copier la partition déchiffrée bit à bit
$ sudo dd bs=1 if=/dev/mapper/partition of=
# let's play !
```

Il ne reste plus qu'à trouver ces données cachées ;) !

### A la recherche des données cachées

C'est là que la partie fun commence... la résolution va grandement dépendre du nombre d'indices que vous aurez pris, de votre chance et d'un peu d'intuition. Sans prendre aucun indice, la tâche s'annonce difficile mais pas impossible ; la partition est relativement petite (20Mo). On peut donc parcourir l'hexadécimal de cette partition assez rapidement, pour cela j'utiliserai [GHex](https://doc.ubuntu-fr.org/ghex).

Ce challenge sert de prétexte pour parler dissimulation de données et système de fichier. Je vais donc détailler des concepts peu utiles pour *résoudre* le challenge mais intéressants si on veut le *comprendre*.

---

Tout d'abord, nous avons à faire à un système de fichier `ext2` (en vrai j'avais fais une partition `ext4` mais je ne suis pas certains que ce détail soit d'une grande importance) :

```bash
$ file partition_dechiffree.dd
Linux rev 1.0 ext2 filesystem data (mounted or unclean), UUID=d5e98c25-8bcb-4567-baac-87b049611a90 (extents) (64bit) (large files) (huge files)
```

### Système de fichier : les bases

Un système de fichier fonctionne **globalement** toujours de la même manière et se compose deux parties :

1. Un index au début qui répertorie tout ou partie des fichiers qui sont présents sur la partition, avec éventuellement leur localisation.

2. La partie qui comporte les fichiers en eux-mêmes, aux endroits indiqués dans l'index.

Le concept est assez simple mais les déclinaisons sont multiples. On peut disposer de plusieurs indexs, il peut contenir davantage de métadonnées, référencer tout ou partie des fichiers, etc... Pour des raisons de performance, la plupart des FS (si ce n'est tous) utilisent la notion de `block`. En fait, un FS n'associe pas un fichier à un offset sur le disque mais à un ou plusieurs `block`. 

Les blocks ont une taille fixe et contiennent la donnée du fichier auquel ils sont alloués. Si un block n'est pas référencé dans l'index, alors c'est qu'il peut être utilisé pour stocker de la donnée.

![Schéma de fonctionnement simplifié d'un système de fichier.](img/blog/superblock-slack-space/schema_fs.png)

De la même manière, l'index a une taille fixe et s'il n'est pas remplit, alors de nouveaux fichiers peuvent être ajoutés. La taille de l'index est calculé par rapport à la taille de la partition.

### Système de fichier : espace vide

Dans le schéma précédent, j'ai mis en évidence les espaces vides (on parlera de `slack space`). En fait un système de fichier dispose **toujours** d'espace vide/disponible, même lorsqu'il est plein. Cependant, il ne voit que des blocks, et pas des octets. Donc si, comme sur le schéma, un fichier n'a pas une taille qui est **exactement** un multiple d'une taille de block, alors il y aura du slack space.

De la même manière, on peut avoir du slack space dans l'index du système de fichier (on parlera de `superblock`).

Donc, si on part du principe que notre système de fichier comporte des données cachées, le slack space pourrait être un endroit intéressant à investiguer !

### Remarque sur le slack space

Ne nous trompons pas d'objectif : utiliser le slack space (en particulier celui des fichiers) pour cacher de la donnée est risqué. A tout moment l'OS peut désalouer le ou les blocks dans lesquels ont a caché de la donnée, les allouer de nouveau et réécrire par-dessus ! **L'objectif n'est pas l'intégrité des données mais la discrétion.**

On pourrait d'ailleurs imaginer des solutions à ce problème :

- un programme qui passerait par-dessus le système de fichier, qui saurait quel block il ne faudrait surtout pas toucher et les protégeraient contre toute modification.

- redonder la donnée cachée, intégrer des codes correcteurs d'erreur pour être le plus résilient possible.

Pour ce challenge, j'ai préféré utiliser le slack space du superblock car c'est bien plus simple à détecter et l'archive cachée est continue en mémoire (imaginer reconstruire un fichier fragmenté entre plusieurs slack space !).

Il existe d'ailleurs des papiers de recherche pour tenter de détecter ce genre de stéganographie ou pour tenter de récupérer de la donnée qui était là de façon légitime avant mais qui subsiste dans le slack space. C'est passionnant !

### Retour au challenge

Maintenant qu'on sait tout ça, on peut retourner au challenge. En parcourant le fichier, on trouve bien l'index :

![Index de la partition, on observe tous les noms des fichiers.](img/blog/superblock-slack-space/ghex1.png)

Si on va assez loin, on va voir des blocks de données, comme celui-ci qui est le premier alloué au fichier `operation_leviathan.pdf` : 

![Le premier bloc](img/blog/superblock-slack-space/ghex2.png)

On commence généralement par l'hypothèse la plus simple : avant de chercher dans le file slack space on va plutôt regarder dans le superblock. Effectivement il y a quelque chose d'étrange :

![Quelque chose d'étrange dans le superblock](img/blog/superblock-slack-space/ghex3.png)

Ceci ressemble à de la donnée d'un fichier. Or, cela n'a rien à faire ici, au vue de l'offset et de la taille du superblock (4096 : cf l'article précédent). On est en présence d'un fichier et le premier indice payant nous indique qu'on cherche une archive dont le magic number a été *légèrement obfusqué* (je vous avais dis de ne pas hésiter à prendre les hints !).

Pour les plus observateurs, l'archive est sous nos yeux mais elle a été un peu modifiée. Le magic number est normalement `PK` mais là on a `QL` : j'ai simplement ajouté 1 aux deux premiers octets.

Enfin, on sait que l'archive fait **610** octets. Mais comment le savoir si on n'a pas voulu prendre le hint ?!

- vous *auriez dû* prendre le hint si vous bloquiez ici

- en lisant la doc du format zip [The structure of a PKZip file](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html) (pas évident je le reconnais)

Finalement, il reste à modifier le magic number et à extraire l'archive :

```bash
dd if=image.dd bs=1 skip=2048 count=610 > archive.zip
```

Et on trouve **enfin** ce qu'on était venu chercher :

> N’utiliser ce document uniquement en cas d’extrême nécessité.
Ne pas manipuler ce document, il est caché dans le système de fichier pour être indétectable. Le manipuler pourrait le corrompre.
Ne pas manipuler la partition pour les mêmes raisons. Le document operation_leviathan.pdf est un leurre pour forcer l’ennemi à se montrer. A la date et l’heure indiquée dans le document, procédez à l’arrestation des espions présents sur les lieux. Si personne ne se montre, c’est que nous ne sommes pas compromis, poursuivez les opérations. Si une ou plusieurs personnes sont avérées être des espions, rendre compte au QG et attendre les ordres. Code de l’opération : talmud FLAG{slack_space_is_the_new_stego}

### Un mot sur l'obfuscation du magic number

**Pourquoi ?** Ce n'est pas (juste) sadique. Si je ne l'avais pas fais, mon challenge avait une *unintended* vraiment simple : utiliser un outil de carving comme scalpel ou foremost.

En fait, ces outils vont chercher sur l'ensemble de la partition des magic number, s'ils en reconnaissent un, ils vont extraire le fichier. Camoufler ces magic number rend ces outils ineficaces !

Note : il est possible que des outils qui ne se basent pas uniquement sur des magic number mais sur la *structure* même d'un fichier puisse quand même retrouver l'archive. Mais si j'avais aussi dû prendre en compte ce cas, le challenge aurait vraiment été trop difficile...
