---
title: "Espace vide & superblock : trifouiller EXT4 pour le fun (1/2)"
date: 2024-07-21T18:25:27+02:00
draft: false
---

## Résumé

Dans cet article, je détaille la création d'un challenge cyber de forensique pour le CTF du Séminaire Facteur Humain 2024 de l'[ENSIBS](https://www-ensibs.univ-ubs.fr/fr/formations/formations/diplome-d-ingenieur-DI/sciences-technologies-sante-STS/diplome-d-ingenieur-cyberdefense-ICYB00_213.html). Il sert de prétexte au bidouillage d'une partition `ext4`. Nous allons voir comment créer une partition chiffrée avec LUKS, comment la monter, la lire, la copier puis la démonter. Cela, dans l'objectif de cacher des données dans les "espaces vides", ou `slack space`. Plus particulièrement, dans le `slack space` du `superblock`, qui est une composante du système de fichier `ext4`. Pour cela, j'utilise l'outil [Fishy](https://github.com/dasec/fishy).

Cet article ne traite pas de la résolution du challenge en tant que tel mais il y a de nombreuses indications à l'intérieur. La partie 2 est disponible ici : [Espace vide & superblock : trifouiller EXT4 pour le fun (2/2)](https://nathan-out.github.io/posts/espace-vide--superblock-trifouiller-ext4-pour-le-fun-2-2/).

## Création du challenge

On prend un support de stockage amovible (type clef USB), comme ça en cas de mauvaise manipulation, on évite d'endommager une partition sur sa machine hôte.

On branche la clef et on vérifie que tout va bien avec `lsblk` : 

```bash
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                   8:0    1  14,4G  0 disk  
└─sda1                8:1    1    XXM  0 part
└─sda2                8:1    1    XXM  0 part
...
└─sdan                8:1    1    XXM  0 part
```

### Nettoyage de la clef

On supprime intégralement toutes les données présentes sur la clef USB (peut prendre un certains temps en fonction de la taille du la clef) :

**Attention : `/dev/sda` est le périphérique correspondant à ma clef USB, chez vous ce peut être un disque interne, externe... Veillez bien à choisir le bon périphérique sinon vous risquez d'effacer le mauvais !**

```bash
# sda est notre clef USB
sudo shred -n1 --random-source=/dev/urandom -z /dev/sda --status=progress
```

### Création d'une partition

On créé une partition qui va ensuite être chiffrée avec LUKS (solution de chiffrement de partition pour Linux). Enfin, on ajoutera un système de fichier `ext4` sur cette dernière et on pourra enfin l'utiliser.

```bash
# ouvre une invite de commande intéractive
sudo fdisk /dev/sda
	n (new)
	p (partition primaire)
	numéro de partition > laisser par défaut
	premier secteur > +20M (pour LUKS il faut une taille de partition d'au moins 16M 
	w (write and save changes)
```

On créera une partition d'au moins 16Mo, sinon la suite risque de ne pas fonctionner. Plus d'information sur le pourquoi ici : [https://superuser.com/questions/1557750/why-does-cryptsetup-fail-with-container-10m-in-size)](https://superuser.com/questions/1557750/why-does-cryptsetup-fail-with-container-10m-in-size)).

On créé une partition avec `n`, on spécifie que c'est une partition primaire avec `p`, on laisse le numéro de partition et le premier secteur par défaut, et enfin, pour le dernier secteur, je renseigne la taille désirée `+20M`.

Si tout s'est bien passé, `lsblk` devrait vous afficher ceci : 

```
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                   8:0    1  14,4G  0 disk  
└─sda1                8:1    1    20M  0 part
```

### Chiffrer la partition avec LUKS

```bash
sudo cryptsetup luksFormat /dev/sda1
```

Le mot de passe pour le challenge est `milou13041982` et donné dans l'énoncé.

### Créer un système de fichier par-dessus LUKS

Je créé ici une partition `ext4` car l'outil que j'utilise pour cacher le fichier dans le `superblock` supporte ce type de système de fichier.

```bash
# on déchiffre la partition puis on créé le FS par-dessus
sudo cryptsetup luksOpen /dev/sda1 <nom_label_de_votre_choix>
sudo mkfs.ext4 /dev/mapper/<nom_label_de_votre_choix>
# je donne les droits à tous les utilisateurs pour plus de praticité
sudo chmod 777 -R /dev/mapper/<nom_label_de_votre_choix>
```

Il ne reste qu'à ouvrir dans un explorateur de fichier `/dev/mapper/<nom_label_de_votre_choix>` et voilà ! On a maintenant notre partition `ext4`, chiffrée avec LUKS, sur laquelle on va pouvoir s'amuser à cacher des choses !

### Copie de la partition déchiffrée

Bien-sûr, puisque la partition est chiffrée, on va cacher des données dans le clair. Pour cela, il faut en faire une copie (image) et c'est sur cette dernière qu'on travaillera :

```bash
dd bs=1 if=/dev/mapper/<nom_label_de_votre_choix> of=image_dechiffree.dd
```

Notez qu'on ne copie pas `/dev/sda1` car ceci est la partition **chiffrée**.

### Superblock slackspace

Le challenge en comporte en réalité deux : 

1. Identifier le système de fichier et le fait que la partition est chiffrée avec LUKS, la déchiffrer (je fournis le mot de passe) et la monter. Niveau facile quoique non trivial lorsqu'on ne l'a jamais fais. Le flag se trouve dans un fichier.

2. Une fois la première étape passée, on explique que l'image du disque semble étrange, qu'une archive zip pourrait bien y être cachée. Il faut analyser la partition avec un éditeur hexadécimal type `GHex` et identifier que le `superblock` comporte des données dans son espace vide (slackspace). Il s'agit d'une archive qui est cachée là, avec un magic number obfusqué, on la dump avec `dd` et un fichier texte se trouve à l'intérieur avec le flag final.

La seconde partie est bien plus difficile. Il faut une connaissance plus fine du système de fichier et les bons outils pour lire les informations bas niveau (taille des blocks etc...). Cette partie requiert également de reconnaître le magic number d'un fichier zip. C'est pourquoi j'ajoute des hints payants qui se débloquent au fur et à mesure (le challenge est sur 100pts) :

0. Le `magic number` de l'archive a été **légèrement** obfusqué pour ne pas que le challenge soit résolvable avec un simple outil de carving type `scalpel` ou `foremost` ;). -0pts (gratuit)

1. Ne faites pas confiance au système de fichier : il pourrait vous *cacher* des choses. -10pts

2. Connaissez-vous les *superblock* en `ext4` ? -10pts

3. `dumpe2fs` donne pas mal d'informations sur la partition, comme la taille des *blocks*, comme ça vous saurez où chercher :) -20pts

4. Toujours pas trouvé ? Mais enfin regardez ce qu'il se cache dans le *superblock* ! N'y a-t-il pas quelque chose d'étrange ? N'ayez pas peur d'utiliser un éditeur hexa type `GHex`. -20pts

5. Il va faloir mettre les mains dans le cambouis... le fichier caché fait 610 octets tout pile -20pts

6. Les blocks de cette partition font 4096 octets (`dumpe2fs test_zip.dd|grep size`). Le `superblock` commence à l'offset 0. Donc il faut chercher dans les 4096 premiers octets, on y remarque la présence d'un fichier à l'offset 2048 (magic number `QL` qui est en fait le magic number obfusqué d'une archive zip `PK` +1). On sait que le fichier fait 610 octets ! On n'a plus qu'à le dump et modifier le magic number : `dd if=image.dd bs=1 skip=2048 count=610 > archive.zip` -10pts

Je donne la taille exacte de l'archive car il n'est pas trivial de la trouver. Cette information aide beaucoup les participants, grâce à ça ils peuvent utiliser `dd` et être sûr d'extraire précisément l'archive.

### Un outil vraiment très pratique

On pourrait avoir une approche naïve et juste ajouter le fichier dans le superblock puisqu'on sait qu'il commence à l'offset 0 et que sa taille est de 4096 octets. On réécrit par-dessus les octets vides et voilà. Je n'ai pas essayé cette approche, j'ai préféré utiliser cet outil : [Fishy](https://github.com/dasec/fishy). Il est très bien documenté et supporte d'autres systèmes de fichier et d'autres techniques de dissimulation. Il est vraiment très intéressant, si le sujet vous intéresse je vous encourage à jouer avec :).

Le `superblock slack space` est assez simple car le fichier est continu en mémoire et se trouve au début de la partition. Une approche plus complexe aurait été de cacher un fichier plus gros avec le `file slack space`.

Une fois l'outil installé (j'ai utilisé un environnement virtuel pour éviter de trop polluer ma machine hôte), on créé le fichier à dissimuler et on l'envoie à l'outil pour qu'il cache le flux d'octet. En effet, l'outil ne semble vouloir cacher que du texte mais on peut ruser :

```bash
dd if=archive.zip bs=1 | sudo fishy -d img_part_dechiffree_avec_fichier_cache.dd superblock_slack -w
```

### Comme si de rien n'était

La dernière étape est de copier l'image, avec le fichier caché à l'intérieur, dans la partition déchiffrée et de démonter la partition (pas besoin de re-chiffrer). Enfin, il reste à copier la partition chiffrée pour avoir le fichier final qu'on donnera aux participants.

```bash
# copie de l'image de la partition avec le fichier caché dans la partition montée
sudo dd bs=1 if=img_part_dechiffree_avec_fichier_cache.dd of=/dev/mapper/<nom_label_de_votre_choix>
# démontage de la partition
sudo umount /mnt/<nom_label_de_votre_choix>
# copie de la partition chiffrée : fichier final du challenge
sudo dd bs=1 if=/dev/sda1 of=final.dd
```

## Remarques

Ceux qui ont déjà fait du [file carving](https://en.wikipedia.org/wiki/File_carving) savent où je veux en venir : **un fichier continu en mémoire peut-il être retrouvé facilement grâce à un outil de `carving` ?** Même lorsque ce fichier ne se trouve pas à un endroit de la partition où on s'attendrait à le trouver ?

La réponse est oui car un outil de `carving` se base sur les `magic number` (sorte d'identifiant de type de fichier), et en aucun cas sur les métadonnées du système de fichier. Voilà pourquoi j'ai ajouté 1 aux deux premiers octets de l'archive ; cela permet de rendre ces outils inutiles !

La partie 2 est disponible ici : [Espace vide & superblock : trifouiller EXT4 pour le fun (2/2)](https://nathan-out.github.io/posts/espace-vide--superblock-trifouiller-ext4-pour-le-fun-2-2/).
