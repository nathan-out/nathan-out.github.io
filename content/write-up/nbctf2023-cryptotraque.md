---
title: "NoBracketsCTF 2023 Write-Up Cryptotraque 1, 2 et 5"
date: 2023-11-20T21:55:22+01:00
draft: false
---

Je détaille ici les corrections des challenges que j'ai créés pour le [NoBracketsCTF](https://nobrackets.fr/) édition 2023. Cette compétition est organisée par le club étudiant [Galette Cidre CTF (GCC)](https://gcc-ensibs.fr/), par des étudiants des spécialités Cyberdéfense, Cybersécurité du Logiciel et Cybersécurité et Sciences des Données de l'Ecole Nationale Supérieure d'Ingénieur de Bretagne Sud (ENSIBS). Elle vise à initier des élève de l'enseignement secondaire à la cybersécurité, en les confrontant à des challenges variés.

Si vous souhaitez tenter de résoudre ces challenges, les fichiers sources sont disponibles sur mon Github : [nathan-out/challs-nbctf-2023](https://github.com/nathan-out/challs-nbctf-2023). Pour chaque challenge, le fichier `challenge.yml` comporte la consigne, les éventuels indices (hints) et la réponse (le flag). Le fichier du challenge se trouve dans `ressources/`. **Les challenges sont indépendants les uns des autres.**

A noter que les challenges Cryptotraque 3, 4 et 4.5 étaient dans la catégories OSINT et créés par Aureliof.

## Cryptotraque 1

Un fichier odt est comme une archive. Partant de là, il suffit d'unzip le fichier odt (ou bien de modifier son extension par zip pour ensuite l'ouvrir comme une archive) et d'aller voir dans le fichier `meta.xml`. La balise `generator` contient les informations requises dans le flag : `LibreOffice/7.5.4.2$Linux_X86_64`.

Flag : NBCTF{LibreOffice/7.5.4.2$Linux_X86_64}

## Cryptotraque 2

Chaque ligne représente une transaction. La première colonne est le wallet (portefeuille) qui a émit une transaction, la seconde est le wallet qui a reçu la transaction et la dernière colonne est le montant de la transaction.

Nous savons également que le wallet qui a reçu la rançon est celui-ci : `9e7e75841c73a0b391a1a0485948c30a`.

Il faut donc suivre les transactions que ce portefeuille a effectué. Par exemple, si A donne envoie à B et B envoie ensuite à C, nous savons que le portefeuille final est C. La logique est similaire ici. 

La difficulté est de comprendre comment s'organisent les informations et ensuite de programmer un algorithme qui va suivre les transactions (ou de le faire à la main pour les plus courageux :p).

### Avez un algo, c'est quand même plus rapide !

Un algorithme serait celui-ci, les explications suivent :

```python
def suivre(adresse_source, fichier):
	f = open(fichier,'r')
	data = f.read()
	data=data.split('\n')
	for ligne in data:
		l = ligne.split(',')
		if l[0] == adresse_source:
			adresse_source = l[1]
	print(adresse_source)

suivre('9e7e75841c73a0b391a1a0485948c30a', 'transactions.csv')
```

La fonction suivre prend en entrée une adresse source (l'adresse du portefeuille qui a reçu la rançon) et le fichier qui contient toutes les transactions. On ouvre le fichier avec la fonction `open` qui prend le nom du fichier et le mode d'ouverture, ici `r` pour "read". En effet, nous n'allons pas écrire dans le fichier mais simplement le lire.

Pour charger les données du fichier dans une variable nous utilisons la fonction `read` qui ne prend pas de paramètres. Ainsi, `data` contient l'entièreté des transactions.

Puisque nous lisons un fichier CSV, chaque ligne se termine par un saut de ligne, ou le caractère `\n`. Ce caractère spécial est celui que vous tapez lorsque vous appuyez sur votre touche "Entrée". `data.split('\n')` va retourner une liste de valeur où chaque élément de la liste est une ligne du fichier. Par exemple :

```csv
ligne1_valeur1,ligne1_valeur2,ligne1_valeur3
ligne2_valeur1,ligne2_valeur2,ligne2_valeur3
```

Contenu de la variable `data` avant le split :

```
data = "ligne1_valeur1,ligne1_valeur2,ligne1_valeur3\nligne2_valeur1,ligne2_valeur2,ligne2_valeur3"
```

Après le split :

```
data = ["ligne1_valeur1,ligne1_valeur2,ligne1_valeur3",
"ligne2_valeur1,ligne2_valeur2,ligne2_valeur3"]
```

La ligne suivante nous permet d'itérer sur la liste `data`, donc sur chaque ligne du fichier CSV. Ensuite, nous "splittons" encore une fois mais sur `,`.

```python
l = ligne.split(',')
```

La variable `l` après le split contient ceci (pour chaque ligne) :

```
l = ["ligne1_valeur1", "ligne1_valeur2", "ligne1_valeur3"]
```

Enfin, si le premier élément de `l` (indice `0`), qui est notre adresse source, est égal à `adresse_source`, alors `adresse_source` devient notre nouvelle adresse à suivre. Cette nouvelle adresse est `l[1]` puisque c'est elle qui a reçu la transaction.

En effet, au premier tour de boucle, `adresse_source` est l'adresse du wallet qui a reçu la rançon, et si l'on trouve quelque part dans le fichier que cette adresse a effectué une transaction, alors il faut suivre cette nouvelle adresse. **En fait, on suit les transaction à partir du wallet qui a reçu la rançon**. Lorsqu'on a parcouru toutes les lignes du fichier, la dernière adresse qui a reçu de l'argent est celle que nous cherchions !

Flag : NBCTF{0d26d74d0daad532ac50852fd5742099}

### Complément

Pour les curieux qui s'intéressent à la façon dont j'ai généré le fichier pour le challenge, je vous laisser aller voir ici : [https://github.com/nathan-out/challs-nbctf-2023/blob/main/cryptotraque2/generateur.py](https://github.com/nathan-out/challs-nbctf-2023/blob/main/cryptotraque2/generateur.py).

Note : on aurait pu utiliser le package Python `csv` qui est prévu pour ouvrir et manipuler des fichiers CSV. Cependant, le traitement est relativement simple et n'aurait rien simplifié ici.

Note 2 : on ne s'intéresse pas au montant de la transaction avec cette méthode. Les plus persipaces auront remarqué qu'il n'y a pas de frais sur les transactions, et qu'il n'y a pas non plus de découpage des fonds ; un wallet transfère toujours tout ce qu'il contient au prochain. Je vous laisse imaginer un nouvel algorithme si maintenant le wallet qui a reçu la rançon la découpe en plus petits montants et les envoie à plusieurs portefeuilles, avant que la raçon reviennent finalement dans un unique portefeuille. On serait dans ce cas plus proche d'un vrai mixeur mais la résolution serait bien plus difficile :).

## Cryptotraque 5

Nous savons que le document contient des informations cachées. Pourtant, en l'ouvrant, nous ne distinguons pas de choses inhabituelles : pas de texte en blanc sur fond blanc, pas de texte écrit en petit, pas d'image qui cacherait du texte...

Le contenu secret ne semble pas être ici. Pour ceux qui ont résolu Cryptotraque1 (ou qui ont lu le wiki), vous savez maintenant qu'un fichier Office peut être vu comme une archive. En ouvrant le fichier avec un gestionnaire d'archive (ex : Winrar), on s'apperçoit qu'il y a d'autres fichiers à l'intérieur.

```
META-INF/
	manifest.xml
Thumbnails/
	thumbnail.png
content.xml
manifest.rdf
meta.xml
mimetype
planification.xml
settings.xml
styles.xml
```

Un technique est d'essayer d'ouvrir successivement tous les fichiers pour voir ce qu'ils contiennent brièvement. On comprend alors un peu mieux comment fonctionnent les fichiers Office : `thumbnail.png` est la miniature, `content.xml` contient le contenu textuel du fichier etc...

Il y a cependant un fichier qui semble différent. En effet, `planification.xml` semble être différent des autres fichiers xml. De plus, il est de loin le plus gros de tous (39ko contre 13.6ko pour le second fichier xml le plus volumineux). La date ne correspond pas avec les autres fichiers. Si vous prenez le temps de lire un peu ce que contient `planification.xml` vous pourrez voir ces quelques mots au tout début du fichier :

```
mimetypeapplication/vnd.oasis.opendocument.textPK
```

Ou bien, en utilisant la commande `file planification.xml` (voir wiki) :

```
planification.xml: OpenDocument Text
```

A ce moment là, soit vous avez déjà compris qu'il s'agit d'un document Open Office, soit une rapide recherche internet vous mettra sur la bonne voie.

En renommant le fichier avec une extension compatible (par exemple odt), ou en ouvrant le fichier avec Open Office, vous avez accès à un tout nouveau document !

A la toute fin du document qui explique la façon dont s'est déroulé le piratage, vous trouverez le flag : `que_pour_largent2023!`.
