---
title: "NoBracketsCTF Finale 2023 Write-Up Broken Zip & Cute kitten"
date: 2023-11-28T11:49:27+01:00
draft: false
---

Je détaille ici les corrections des challenges **Broken Zip** et **Cute Kitten** que j'ai créés pour la finale du [NoBracketsCTF](https://nobrackets.fr/) édition 2023. Cette compétition est organisée par le club étudiant [Galette Cidre CTF (GCC)](https://gcc-ensibs.fr/), par des étudiants des spécialités Cyberdéfense, Cybersécurité du Logiciel et Cybersécurité et Sciences des Données de l'Ecole Nationale Supérieure d'Ingénieur de Bretagne Sud (ENSIBS). Elle vise à initier des élève de l'enseignement secondaire à la cybersécurité, en les confrontant à des challenges variés. La finale s'est déroulée au Couvent des Jacobins à Rennes à l'occasion de l'[European Cyber Week (ECW)](https://www.european-cyber-week.eu/).

Si vous souhaitez tenter de résoudre ces challenges, les fichiers sources sont disponibles sur mon Github : [nathan-out/challs-nbctf-2023](https://github.com/nathan-out/challs-nbctf-2023) **sauf pour la ressource associée à Cute Kitten qui est disponible ici** : [https://drive.google.com/file/d/1iNXVh9YicXsu_tbmTvJ2MWkilx88xbak/view?usp=sharing](https://drive.google.com/file/d/1iNXVh9YicXsu_tbmTvJ2MWkilx88xbak/view?usp=sharing). Pour chaque challenge, le fichier `challenge.yml` comporte la consigne, les éventuels indices (hints) et la réponse (le flag). Le fichier du challenge se trouve dans `ressources/`. **Les challenges sont indépendants les uns des autres.**

![Logo du NoBracketsCTF.](/img/write-up/nobracketsctf_finale.jpg)

<figcaption>Le Couvent des Jacobins, place Saint-Anne à Rennes a accueilli la finale du NoBracketsCTF 2023.</figcaption>

## Broken Zip

On sait que le fichier est une archive zip endommagée et chiffrée avec mot de passe. On peut s'en convaincre en essayent de l'ouvrir avec WinRar :

> L'en-tête de fichier est corrompu : secret.txt

Ce message signifie que le début du fichier, l'en-tête, n'est pas correcte. L'en-tête regroupe plusieurs informations, le plus souvent peuvent y figurer le nom du fichier, sa taille, une somme de contrôle pour s'assurer de l'intégrité du fichier etc...

Chaque en-tête a un format bien défini par une norme. Ce site nous donne de précieuses informations sur la façon dont est codée l'en-tête d'une archive zip : [https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html).

Nous pouvons supposer que l'en-tête de `secret.zip` n'est pas conforme à ce qui est énoncé ici. Pour observer le fichier sous sa forme hexadécimale, ouvrez le fichier avec [HxD](https://mh-nexus.de/en/hxd/). En regardant les 4 premiers octets du fichier, la signature, on se rend compte que ceux-ci sont corrects : `50 4B 03 04`. Continuons notre investigation sur les octets suivants :

- `14 00` : cela semble être la bonne version, puisque c'est la même que sur le site.

- `09 00` : il semble ici y avoir un problème puisque la valeur correspondrait à `unused` alors que le fichier est chiffré. Continuons de vérifier un par un les champs et nous reviendrons dessus ensuite si nous n'avons rien trouvé.

- `63 00` : (compression method) arrêtons-nous un moment sur cette valeur.

Le site donne la signification des valeurs **en décimal**, 0x63=99. Cette valeur n'est pas indiquée sur le site, en cherchant "zip compression method 99" on découvre que cette valeur correspond à un chiffrement avec AES. Cela semble cohérent, le fichier est protégé par un mot de passe et donc chiffré. Encore raté ! Poursuivons...

- viennent ensuite `File modification time` et `File modification date`, cela ne semble pas bloquant pour une ouverture de fichier. Si vous essayez de les décoder, vous verrez que ces sont des dates cohérentes.

- `D4 0B 70 73` le CRC32-checksum. Ceci sert à s'assurer que le fichier est intègre. Nous pouvons supposer que ce checksum n'est pas bon mais continuons d'avancer, comprendre comment et sur quelles données le CRC32 est calculé n'est pas simple, essayons d'abord de voir si les champs suivants sont correctes.

- `4E 00 00 00 32 00 00 00` : `compressed` et `uncompressed size`. Assumons que ces valeurs sont correctes, c'est un petit fichier cela ne semble pas incohérent. Nous reviendrons dessus si le dernier champ ne donne rien non plus.

- `0F 14` : `File name length`, ici il y a quelque chose de louche. On peut voir dans les octets suivants le nom du fichier contenu dans l'archive (qui est d'ailleurs le même que le nom de l'archive). La longueur du nom du fichier est de 10 caractères. Or ici, on voit bien que la valeur qui est donnée est très nettement supérieure ! Il semble donc y avoir un problème sur ce champ !

10 en hexadécimal donne 0xA. Si on remplace `0F 14` par `00 0A` cela ne fonctionne pas le fichier est toujours corrompu, pourquoi ?

En descendant plus loin sur le site, on a un exemple d'un fichier qui s'appelle `file1` et les octets qui codent la taille de son nom sont `05 00` : **les octets sont lus à l'envers !**

En remplacant par `0A 00` on peut rentrer le mot de passe et le fichier s'ouvre !

## Cute Kitten

L'énoncé nous informe que le suspect détiendrait des *images* de chatons. Dans un premier temps, il faut identifier où pourrait se trouver un pareil fichier. Nous sommes face à un **dump RAM** qui correspond à une "photographie" de la mémoire vive de l'ordinateur. La première chose à faire est d'identifier l'outil adapté : Volatility.

Pour simplifier le challenge, j'ai indiqué qu'il fallait utiliser **Volatility 2.5.0**. C'est un outil en Python3 qui permet d'extraire des informations de la RAM.

### Manière attendue par le créateur

Initialement, j'ai créé ce challenge pour que vous suiviez ces étapes pour le résoudre : 

1. visualisation des processus avec `pstree`, `pslist` ou `psscan`.

2. identification du processus qui pourrait utiliser une image (Windows Photo).

3. identification des *handle* du processus qui pourrait servir à visualiser la photo.

4. récupération de l'adresse virtuelle ou physique du *handle* du processus.

5. extraction du fichier avec `dumpfiles`. Le flag est ensuite écrit sur la photo.

Le soucis étant qu'il n'est pas possible d'extraire l'image avec `dumpfiles` sous Volatility3. **Si vous y êtes arrivés par un autre moyen je serai très curieux de savoir comment !**

### Une méthode qui fonctionne

#### Identifier les processus d'intérêt

La première chose à faire est d'identifier les processus qui tournaient sur l'ordinateur au moment où le dump RAM a été effectué. Pour cela, on peut utiliser le plugin `pstree`, `pslist` ou `psscan`.

```
python.exe D:\volatility3-2.5.0\volatility3-2.5.0\vol.py -f .\memdump\memdump.mem windows.psscan
```

A noter que la première exécution peut prendre du temps du fait du téléchargement d'un fichier requis pour l'analyse de ce dump (le fichier PDB correspondant pour le build du Windows10).

On obtient ceci (tronqué) :

```
PID	PPID	ImageFileName	Offset(V)	Threads	Handles	SessionId	Wow64	CreateTime	ExitTime	File output

4	0	System	0xdf02d2883040	113	-	N/A	False	2023-11-17 13:11:56.000000 	N/A	Disabled
92	4	Registry	0xdf02d29ba040	4	-	N/A	False	2023-11-17 13:11:53.000000 	N/A	Disabled
324	4	smss.exe	0xdf02d3166080	2	-	N/A	False	2023-11-17 13:11:56.000000 	N/A	Disabled
648	528	services.exe	0xdf02d3838340	5	-	0	False	2023-11-17 13:12:03.000000 	N/A	Disabled
416	404	csrss.exe	0xdf02d3864080	10	-	0	False	2023-11-17 13:12:01.000000 	N/A	Disabled
1484	648	svchost.exe	0xdf02d389e2c0	1	-	0	False	2023-11-17 13:12:20.000000 	N/A	Disabled
...
2400	5952	msedge.exe	0xdf02d7724080	14	-	2	False	2023-11-17 14:35:50.000000 	N/A	Disabled
7040	5952	msedge.exe	0xdf02d772e080	14	-	2	False	2023-11-17 14:34:25.000000 	N/A	Disabled
2220	5952	msedge.exe	0xdf02d7730080	13	-	2	False	2023-11-17 13:44:15.000000 	N/A	Disabled
6692	788	SearchApp.exe	0xdf02d77e3080	31	-	2	False	2023-11-17 13:19:02.000000 	N/A	Disabled
4804	5952	msedge.exe	0xdf02d77ea080	12	-	2	False	2023-11-17 14:35:50.000000 	N/A	Disabled
6204	5952	msedge.exe	0xdf02d77fb080	9	-	2	False	2023-11-17 13:44:04.000000 	N/A	Disabled
...
1116	1704	MicrosoftEdgeU	0xdf02d8727300	3	-	0	True	2023-11-17 13:13:17.000000 	N/A	Disabled
...
8140	788	Microsoft.Phot	0xdf02d98e3080	21	-	2	False	2023-11-17 14:32:20.000000 	N/A	Disabled
...
2676	5952	msedge.exe	0xdf02da2e1080	13	-	2	False	2023-11-17 13:44:16.000000 	N/A	Disabled
6096	5952	msedge.exe	0xdf02da38f080	16	-	2	False	2023-11-17 14:35:23.000000 	N/A	Disabled
6536	5952	msedge.exe	0xdf02da570080	14	-	2	False	2023-11-17 13:44:14.000000 	N/A	Disabled
4796	5952	msedge.exe	0xdf02da571080	13	-	2	False	2023-11-17 13:44:15.000000 	N/A	Disabled
1800	5952	msedge.exe	0xdf02da573080	13	-	2	False	2023-11-17 13:44:15.000000 	N/A	Disabled
1672	5952	msedge.exe	0xdf02da5ca080	12	-	2	False	2023-11-17 14:35:56.000000 	N/A	Disabled
...
```

On observe un processus qui pourrait nous intéresser : `Microsoft.Phot` (le nom est tronqué). En se renseignant sur ce processus, on trouve qu'il s'agit de la visionneuse d'image de Windows. On observe également des processus relatifs à un navigateur web (`msedge.exe`, `MicrosoftEdgeU`) qui pourraient être intéressants car ils peuvent aussi être utilisés pour des images. En résumé, les PID (identifiant d'un processus) intéressants pour nous sont :

- 8140 (`Microsoft.Phot`), visionneuse photo windows

- 5952 (`msedge.exe`), processus père de la plupart des autres processus du même type

- et sûrement d'autres comme `svchost.exe` (je vous laisse vous renseigner pourquoi ;))

#### Identifier les fichiers chargés

Une autre méthode serait de lister tous les fichiers utilisés par le système avec `windows.filescan`. Ce plug-in va nous renseigner sur les fichiers ouverts par le système. Je redirige la sorie vers un fichier texte pour que je puisse plus facilement chercher des fichiers de type image. A noter que cette méthode est assez imprécise, d'autant qu'on dispose déjà des PID des processus d'intérêts.

```
python.exe D:\volatility3-2.5.0\volatility3-2.5.0\vol.py -f .\memdump\memdump.mem windows.filescan > filescan
```

Le fichier est très volumineux : un système ouvre beaucoup de fichiers ! Rappellons-nous que nous sommes à la recherche d'une **image**, donc on peut chercher pour des fichiers aux extensions telles que png, jpg, jpeg, bmp...

Il n'y a pas de fichiers png, ni jpg, ni bmp mais surprise, on trouve un fichier jpeg :

```
Offset	Name	Size
0xdf02daf09710	\Users\Greg\Downloads\gGR3ksH.jpeg	216
```

#### Extraction de la mémoire du processus

Maintenant que nous avons identifié le fichier `gGR3ksH.jpeg` et plusieurs processus d'intérêt, tentons de voir ce qu'ils contiennent. Pour cela, on peut extraire la mémoire d'un processus (ici `msedge.exe`) grâce au plug-in `memmap` : 

```
python.exe D:\volatility3-2.5.0\volatility3-2.5.0\vol.py -f .\memdump\memdump.mem -o "dump-5952/" windows.memmap --dump --pid 5952
```

Nous avons donc un dump de la mémoire du navigateur web ouvert lors de l'acquisition et nous recherchons une image, et si le navigateur web contient également le lien vers cette image ? Un moyen rapide de trouver des informations de ce dump est d'extraire les chaînes de caractères présentent dans le fichier avec `strings` :

```
strings.exe pid.5952.dmp > 5952-strings
```

Là encore, il y a beaucoup de données mais nous savons ce que nous cherchons ! Cherchez le nom du fichier et vous trouverez un lien vers le site qui héberge l'image : `https://i.imgur.com/gGR3ksH.jpeg`. Le flag est écrit sur l'image.

#### Bonus

Pour les curieux, on peut également lister les *handles* (ressources utilisées) d'un processus spécifique (ici la visionneuse photo Windows) : 

```
python.exe D:\volatility3-2.5.0\volatility3-2.5.0\vol.py -f .\memdump\memdump.mem windows.handles --pid 8140
```

```
8140    Microsoft.Phot  0xdf02db4903a0  0x12c0  File    0x120089        \Device\HarddiskVolume2\Users\Greg\Downloads\gGR3ksH.jpeg
```

Malheureusement, on ne trouve rien de plus que des noms, pas le lien. Cependant, cela confirme que l'utilisateur avait le fichier `gGR3ksH.jpeg` d'ouvert sur son PC. Etant donné son nom peu commun, on peut légitimement pensé que l'utilisateur a téléchargé ce fichier spécifiquement, puis l'a ouvert.
