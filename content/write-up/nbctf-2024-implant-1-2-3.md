---
title: "NoBracketsCTF 2024 Write-Up Implant 1, 2 et 3"
date: 2024-10-13T19:00:06+03:00
draft: false
---

Write-Up des 3 challenges de forensique "Implant 1, 2 et 3". Les fichiers sources sont disponibles ici :

- [Part 1 - Amcache.zip](https://nathan-out.github.io/img/content/nbctf2024/Amcache.zip)
- [Part 2 - logs.zip](https://nathan-out.github.io/img/content/nbctf2024/logs.zip)
- [Part 3 - capture_reseau.zip](https://nathan-out.github.io/img/content/nbctf2024/capture_reseau.zip)

## Implan 1/3

Rappel du challenge : 

>  `noclick`, un de nos spécialiste drone n'a plus donné signe de vie depuis plusieurs jours. Il travaillait activement sur un "exploit" (programme destiné à exploiter une faille) pour neutraliser la reconnaissance faciale des drones du gouvernement. On pense qu'il aurait été arrêté ! Un informateur s'est rendu à son domicile qui était vide. Sur nos ordres, il a lancé un programme qui a collecté toutes les traces d'activité sur le poste de `noclick`. Ces traces vont vous permettre de reconstituer ce qu'il s'est passé sur sa machine. Il nous faut tout d'abord identifier le programme malveillant. L'informateur pense que le nom du virus qui a compromis l'ordinateur de `noclick` se trouve dans le fichier joint. Il s'agit d'une extraction de l'`Amcache` (plus d'information à ce propos sur le [wiki](https://wiki.nobrackets.fr/docs/intro), partie Forensic > Windows).
  
L'objectif est de trouver le **nom** du programme malveillant ainsi que son **hash** `SHA1`. Le format du flag est le suivant : NBCTF{<nom_du_programme>-<hash SHA1>}.

### Amcache

Pour reprendre les mots du [wiki](https://wiki.nobrackets.fr/docs/intro) (eux-mêmes tirés du [poster du SANS - Windows Forensic Analysis](https://cyber-ssct.com/SANS%20Digital%20Forensic%20Poster.pdf)), l'Amcache *"référence les applications installées, les programmes exécutés (ou présents sur la machine), les pilotes chargés, etc. Ce qui distingue cet artefact, c'est qu'il référence également le hash des applications et des pilotes."*. Nativement, l'Amcache est une base de données au format **binaire** (située ici `C:\Windows\AppCompat\Programs\Amcache.hve`). C'est-à-dire qu'elle n'est pas lisible tel quel (essayez de l'ouvrir avec le bloc note et vous comprendrez). Ce qu'on a ici est une **traduction de l'Amcache au format textuel**.

Dans la "vraie vie", on travaille avec des outils qui font cette traduction automatiquement. Pour information, j'ai utilisé [KAPE](https://www.kroll.com/en/services/cyber-risk/incident-response-litigation-support/kroll-artifact-parser-extractor-kape) qui est un des outils de référence en la matière. Si vous voulez apprendre à l'utiliser, [TryHackMe propose un cours gratuit pour apprendre à utiliser KAPE](https://tryhackme.com/r/room/kape), je vous le recommande lui et tous les autres cours de la plateforme.

### Analyse

Armé de ces nouvelles connaissances, comment identifier un programme malveillant ? Commençons par comprendre les données que l'Amcache contient. Chaque colonne a une signification bien précise mais je ne vais en détailler qu'une partie : 

- `FileKeyLastWriteTimestamp` : horodatage de l'écriture de la ligne (on peut raisonnablement l'interprêter comme la date de **dernière** exécution du programme/driver [en vrai c'est un peu plus compliqué mais on ne rentrera pas dans le détail ici]).

- `SHA1` : le **hash** (au format SHA1) du programme/driver. Si vous ne savez pas ce qu'est un hash, référez-vous au [wiki](https://wiki.nobrackets.fr/docs/intro). Pour faire simple, il s'agit d'une chaîne de caractère qui nous permet d'identifier de façon unique le programme/driver.

- `FullPath` : où est situé le programme/driver sur la machine.

On peut encore obtenir bien d'autres informations comme la taille du programme/driver, sa version, sa langue... mais nous n'en aurons pas besoin ici.

On sait que `noclick` travaillait sur un programme relatif à des **drones**. On peut donc commencer par là et chercher les occurences de ce mot dans le fichier CSV, que vous pouvez ouvrir avec un bloc note, Excel, LibreOffice Calc ou encore Google Sheet. On aurait pu aussi se demander si un programme situé dans le dossier téléchargement apparaissait.

On tombe sur une seule occurence dont voici les informations :

- `FileKeyLastWriteTimestamp` : `2024-08-19 18:05:56.8477053`

- `SHA1` : `89542eac72f11afb0711535d0404c79aad9c3728`

- `FullPath` : `c:\users\noclick\downloads\...important_nouvel_exploit_drone_a_tester.exe`

On s'apperçoit alors que le fichier exécutable se trouve dans le dossier `downloads` ce qui corrobore les dires de l'informateur. Les plus expérimentés d'entre-vous seraient tentés de rechercher le hash sur [VirusTotal](https://www.virustotal.com/) qui est une base de données spécialement conçue pour référencer les malwares. Malheureusement, étant donné que j'ai développé moi-même le malware et que je ne l'ai pas mis sur VirusTotal, il n'apparaît pas. Néanmoins, en fouillant un peu plus le fichier CSV, on ne trouve pas d'autres programmes exécutés depuis le dossier de téléchargement et chaque autre programme semble être cohérent avec un Windows. 

### Références

- [Wiki du NoBracketsCTF](https://wiki.nobrackets.fr/docs/intro)

- [Cours TryHackMe gratuit sur KAPE](https://www.kroll.com/en/services/cyber-risk/incident-response-litigation-support/kroll-artifact-parser-extractor-kape)

- [SANS - Windows Forensic Analysis Poster](https://cyber-ssct.com/SANS%20Digital%20Forensic%20Poster.pdf)

- [Analyse de l'AmCache - ANSSI](https://cyber.gouv.fr/publications/analyse-de-lamcache)

## Implant 2/3

Rappel du challenge : 

> L'informateur vous a envoyé les `logs` de l'ordinateur de `noclick`. Ce sont des fichiers qui répertorient les actions qui se sont déroulées sur le système (plus d'informations sur le [wiki](https://wiki.nobrackets.fr/docs/intro)). Quand le virus a-t-il été lancé ? Quel est le **processus père** (*parent process*) qui a lancé le malware ? Le flag est au format NBCTF{<date et heure du lancement du virus>-<nom processus père>}.

### Logs Windows

De la même manière que pour la partie 1, les logs Windows ont été traduits dans un format textuel. On a donc un fichier CSV que vous pouvez ouvrir avec votre bloc note, LibreOffice Calc, Excel ou encore Google Sheet. Je déconseille le bloc note car il n'est pas adapté pour les CSV. L'analyse sera bien plus simple si vous prenez un tableur.

#### Quelques informations sur les logs

On peut voir les logs comme un journal de bord du système. Windows garde trace des différentes actions telles que la connexion à un réseau, les programmes lancés, leurs actions sur le système, l'ouverture de fichier etc... Windows fournit également un outil graphique qui permet de les consulter, les trier, effectuer des recherches... 

> On peut même ajouter des sources de logs à Windows. Dans ce challenge, j'ai ajouté [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) qui permet d'avoir des logs plus précis. En particulier, il permet de voir davantage d'actions que réalisent un programme. Il **n'est pas** installé de base sous Windows mais il s'agit d'un *must have* lorsqu'on souhaite monitorer des machines. Pour les intéressé(e)s, je vous invite à vous renseigner sur cet outil.

Les logs sont très riches, il n'est pas facile de s'y reprérer pour un débutant. Néanmoins, les noms des colonnes peuvent aider à comprendre. Ici, nous allons nous intéresser aux colonnes suivantes : 

- `TimeCreated` : horodatage du log en question.

- `EventID` : il s'agit d'un identifiant qui permet... d'identifier un événement particulier. Par exemple, `4624` signifie *an account was successfully loged on*. C'est une donnée très utile mais qui ne sera pas utilisée dans ce challenge. *Nota* : le champ `MapDescription` est la description associée à l'eventID.

- `UserName` : quel *utilisateur* est à l'origine de l'action (du log). On peut retrouver l'utilisateur "humain" qui utilisait l'ordinateur, auquel cas il sera noté comme suit : `WINDOWS10\<username>` (dans notre cas l'*username* sera `noclick`).

> On peut également voir des utilisateurs comme `AUTHORITE NT\Système`, `AUTHORITE NT\SERVICE RESEAU`. Dans ce cas, cela signifie que ce sont des actions que le système effectue automatiquement. Une action effectuée par `AUTHORITE NT\Système` ne signifie **pas forcément** qu'il s'agit d'une activité malveillante. Dans ce challenge, on n'ira pas dans le détail.

- `PayloadData4` : contient parfois des informations, en l'occurrence le *parent process* (processus père) qui a lancé le processus dont il est question dans le log.

- `ExecutableInfo` : le fichier `exe` à l'origine du log.

- `Payload` : contient la majorité des informations croustillantes qu'on détaillera dans la partie "*Bonus : résolvez la partie 3 grâce au payload*". **N'est pas obligatoire pour ce challenge.**

#### Processus père et fils

Un processus est un programme lancé. Quand vous double-cliquez sur un programme il est chargé en mémoire (RAM) et Windows créé un processus. Souvent, un programme est lancé par un autre programme. Si `programme1.exe` lance `programme2.exe`, alors `programme1.exe` est le processus **père** de `programme2.exe`. Inversement, `programme2.exe` est le processus **fils** de `programme1.exe`.

On serait tenté de dire que, si je lance un programme, alors il n'a pas de processus père. En réalité, Windows a un fonctionnement bien précis et chaque programme possède un processus père (pour faire simple). Si vous voulez comprendre en détail le fonctionnement de Windows sur cette partie, je vous invite à consulter ce poster du SANS : [Find Evil - Know Normal - SANS](https://share.ialab.dsu.edu/cae_workshops/2019/Incident%20Response/Supplementary%20Material/SANS_Poster_2018_Hunt_Evil_FINAL.pdf).

### Investiguons !

Armé de toutes ces connaissances, nous pouvons commencer l'investigation. A ce stade, nous savons que le programme malveillant s'appelle `important_nouvel_exploit_drone_a_tester.exe`. Une recherche de ce programme dans les logs nous sort ces informations : 

- `TimeCreated` : 2024-09-08 15:20:39.3203769

- `EventID` associé à `MapDescription` pour rendre le tout compréhensible : 1 (*Process creation*).

- `UserName` : `WINDOWS10\noclick` cela indique que c'est l'utilisateur de la machine qui est à l'origine de cette action.

- `ParrentProcess` : `C:\Windows\System32\cmd.exe` le processus qui a lancé `important_nouvel_exploit_drone_a_tester.exe`.

- `ExecutableInfo` : `important_nouvel_exploit_drone_a_tester.exe` le programme à l'origine du log.

(On s'intéressera au champ `Payload` dans la partie suivante, en bonus)

D'après ces informations, on peut en tirer la conclusion suivante : 

> Le **2024-09-08** à **15:20:39**, l'utilisateur **noclick** a exécuté le programme `important_nouvel_exploit_drone_a_tester.exe` (`EventID` 1 (*Process creation*)) **depuis un invite de commande** (`ParrentProcess` `cmd.exe`).

Ici, il s'agit uniquement de la première occurence, donc le log **le plus vieux**. Pas besoin de s'intéresser aux autres pour ce challenge car ce sont les actions qu'effectue le programme ensuite. Ce qui nous intéresse ici est de savoir quand le malware a été exécuté pour la première fois et par quel programme.

### Bonus : résolvez la partie 3 grâce au payload

Cette partie est pour les participant(e)s déjà un peu expérimenté(e)s. Si vous voulez comprendre plus en détail, vous êtes au bon endroit. Les plus curieux(ses) auront peut-être remarqué des actions étranges... En particulier dans la colonne `Payload`. Cette dernière contient beaucoup d'informations au format `JSON`. C'est assez difficile à lire mais on peut utiliser un [JSON formatter en ligne](https://jsonformatter.org/) qui va effectuer des sauts de lignes ainsi que des indentations pour rendre la lecture plus simple (j'ai enlevé ce qui ne nous intéresse pas) : 

```json
{
  "EventData": {
    "Data": [
      {
        "@Name": "UtcTime",
        "#text": "2024-09-08 15:20:39.216"
      },
      {
        "@Name": "Image",
        "#text": "C:\\Users\\noclick\\Downloads\\IMPORTANT_nouvel_exploit_drone_a_tester\\IMPORTANT_nouvel_exploit_drone_a_tester.exe"
      },
      {
        "@Name": "CommandLine",
        "#text": "IMPORTANT_nouvel_exploit_drone_a_tester.exe"
      },
      {
        "@Name": "CurrentDirectory",
        "#text": "C:\\Users\\noclick\\Downloads\\IMPORTANT_nouvel_exploit_drone_a_tester\\"
      },
      {
        "@Name": "User",
        "#text": "WINDOWS10\\noclick"
      },
      {
        "@Name": "ParentImage",
        "#text": "C:\\Windows\\System32\\cmd.exe"
      },
      {
        "@Name": "ParentCommandLine",
        "#text": "\"C:\\Windows\\system32\\cmd.exe\" "
      },
      {
        "@Name": "ParentUser",
        "#text": "WINDOWS10\\noclick"
      }
    ]
  }
}
```

On se rend compte ici que cette colonne a elle toute seule nous permet d'avoir toutes les informations nécessaires à la résolution du challenge. Et si on regarde d'autres occurences de notre malware, on tombe sur un *payload* intriguant :

```json
{
  "EventData": {
    "Data": [
      {
        "@Name": "UtcTime",
        "#text": "2024-09-08 15:20:47.704"
      },
      {
        "@Name": "OriginalFileName",
        "#text": "Cmd.Exe"
      },
      {
        "@Name": "CommandLine",
        "#text": "C:\\Windows\\system32\\cmd.exe /c \"dir\""
      },
      {
        "@Name": "CurrentDirectory",
        "#text": "C:\\Users\\noclick\\Downloads\\IMPORTANT_nouvel_exploit_drone_a_tester\\"
      },
      {
        "@Name": "User",
        "#text": "WINDOWS10\\noclick"
      },
      {
        "@Name": "ParentImage",
        "#text": "C:\\Users\\noclick\\Downloads\\IMPORTANT_nouvel_exploit_drone_a_tester\\IMPORTANT_nouvel_exploit_drone_a_tester.exe"
      },
      {
        "@Name": "ParentCommandLine",
        "#text": "IMPORTANT_nouvel_exploit_drone_a_tester.exe"
      }
    ]
  }
}
```

Ne trouvez-vous pas le champ `CommandLine` étrange ? Ce que nous dit ce log c'est que `cmd.exe` a un processus père qui est `IMPORTANT_nouvel_exploit_drone_a_tester.exe`. De plus, `cmd.exe` exécute une commande qui est `dir`, qui sert à lister les éléments dans le dossier courant. Pour le dire autrement, le soit-disant programme qui servirait à exploiter une vulnérabilité dans les drones exécute des commandes afin de savoir ce qui se trouve dans un dossier. Plus étrange encore, lorsqu'on continue de chercher un peu, on tombe sur les commandes suivantes : 

- `cd C:\` (`cd` sert à se déplacer dans les dossiers)

- `type "C:\Users\noclick\Desktop\message du collectif\drone exploit.py"` (`type` sert aficher le contenu d'un fichier)

Moi je trouve que ça ressemble fortement à un programme qui permet à une personne de se balader sur l'ordinateur de `noclick` et de voir le contenu des fichiers... bref, un **implant** !

### Références

- [Cours TryHackMe (malheureusement payant) sur les logs Windows](https://tryhackme.com/room/windowseventlogs)

- [Wiki du NoBracketsCTF](https://wiki.nobrackets.fr/docs/intro)

- [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)

- [Find Evil - Know Normal - SANS](https://share.ialab.dsu.edu/cae_workshops/2019/Incident%20Response/Supplementary%20Material/SANS_Poster_2018_Hunt_Evil_FINAL.pdf)

## Implant 3/3

Rappel du challenge : 

> Récapitulons : `IMPORTANT_nouvel_exploit_drone_a_tester.exe` est notre malware qui a été téléchargé par `noclick`. Il l'a lancé le `2024-09-08 15:20:39` par le biais de `cmd.exe` (ce qu'il a vraiment fait c'est de lancer le virus **depuis** le terminal (`cmd.exe`)). Mais... que fait vraiment ce virus ? Quelles sont ses capacités ? L'informateur vous fournit un fichier qui contient l'activité réseau de l'ordinateur de `noclick` au moment de l'infection. Vous devez identifier les **commandes** que le pirate a exécuté sur la machine de `noclick`.

Un indice gratuit donne l'adresse IP de `noclick` : `172.20.10.3`.

### Pourquoi un dump réseau ?

L'idée est d'observer ce que l'attaquant a fait pendant qu'il contrôlait l'ordinateur de `noclick`. Si vous avez lu la partie bonus de la partie 2, vous avez déjà une idée. Pour cela, rien de mieux que d'enregistrer ce qu'il se passe sur le réseau car, généralement, un malware communique avec son `C2` : "*command and control*". C'est là que l'attaquant se cache et c'est depuis ce serveur distant, situé quelques part sur Internet, qu'il peut piloter son malware. En analysant l'état du réseau au moment de l'infection, on pourra se faire une idée très précise des actions effectuées par le malware.

> En réalite, le C2 ne se trouvait pas sur Internet mais sur un réseau **privé**. Cela pour simplifier la création du challenge. C'est annecdotique ici mais toujours bon de le préciser. En fait, la machine infectée était une machine virtuelle et communiquait avec la machine hôte (machine de l'attaquant : le C2).

### A l'aide, je ne comprends rien !

Pas de panique. Le challenge était volontairement plus difficile que les deux parties précédentes. Ce qu'il se passe sur un réseau est par nature éphémère ; une fois que la donnée est passée d'une machine à une autre, elle est passée. C'est pour cela qu'ici vous avez une **sauvegarde** de tout ce qu'il s'est passé sur le réseau pendant environ 2 minutes.

Il se passe des tonnes de choses sur un réseau, à différents niveaux. Des informations sont échangées continuellement entre différentes machines/programmes. Afin de formaliser la façon dont deux machines s'échangent de la donnée, on utilise des **protocoles**. Un protocole sert à définir, à l'avance, le format des données, leurs type, ce qu'elles signifient etc... Vous connaissez sûrement les plus connus : HTTP, TCP/IP, DNS...

Pour simplifier le challenge, j'ai filtré uniquement le protocole **TCP** (car le malware communique avec son C2 en utilisant TCP). Si je n'avais pas fait cela, vous auriez vu des requêtes HTTP, DNS etc... ce qui aurait fait autant de fausses pistes et de bruits parasites.

### Wireshark ?

La manière la plus commune d'analyser un dump réseau est d'utiliser [Wireshark](https://www.wireshark.org/download.html). C'est un outil très puissant et complet. Vous trouverez des tas de tutoriels en ligne. Ici, on va s'intéresser aux fonctionnalités les plus basiques, à savoir le **filtrage** et le **suivi de flux**.

#### Ce qu'envoi nofix

On émet l'hypothèse que la machine de `noclick` communique avec une machine distante : le C2. Puisqu'on connaît l'IP de `noclick`, on peut filtrer **tout ce qu'envoie sa machine**. L'idée est d'identifier l'adresse IP du C2 afin de se débarasser du bruit.

![Filtrage de ce qui provient de la machine de `nofix` grâce à une requête (barre verte).](/img/write-up/nbctf2024/ip_src_nofix.png)

<figcaption>Filtrage de ce qui provient de la machine de `nofix` grâce à une requête (barre verte).</figcaption>

Bon... je ne sais pas pour vous, mais je ne vois rien d'étrange à première vue. Il y a pas mal d'adresses IP différentes. De plus, lorsqu'on clique sur un paquet, on tombe parfois sur du texte, parfois sur un charabiat incompréhensible (coin inférieur droit de l'écran).

> Wireshark essaie de traduire le binaire en lettres lorsqu'il le peut. Parfois, quand les paquets contiennent du texte, cela peut nous être utile. Néanmoins, le plus souvent, les données sont en binaire.

C'est ici que la difficulté se trouve : il faut démêler et comprendre les différents paquets réseau et fouiller. Cela peut être une bonne idée mais je vais vous fournir un outil très intéressant dans ce cas de figure et assez peu connu.

#### NetworkMiner à la rescousse !

[NetworkMiner](TODO) est un outil qui permet de visualiser autrement une capture réseau. Là où [Wireshark](https://www.wireshark.org/download.html) vous montre les **paquets**, [NetworkMiner](TODO) vous montre les **machines**. Or, c'est justement de cela dont nous avons besoin.

> Pour utiliser [NetworkMiner](TODO) avec le fichier fournit, il faudra le convertir en `.pcap`. Je vous laisse chercher par vous-même comment le faire (on peut le faire depuis [Wireshark](https://www.wireshark.org/download.html)).

Lorsqu'on ouvre le dump réseau, on tombe sur cet interface et tout devient plus clair !

![Toutes les IP de chaque machine qui figure dans le dump réseau.](/img/write-up/nbctf2024/network_miner.png)

<figcaption>Toutes les IP de chaque machine qui figure dans le dump réseau.</figcaption>

Si ce n'est pas clair, pas de panique je vais vous expliquer. On voit ici pas mal d'adresses IP et des noms de domaine. Bon, on peut se dire que les noms de domaines sont légitimes, qu'ils n'entrent pas en compte dans l'attaque. Souvenons-nous de ce qu'on cherche : une machine qui communique avec celle de `nofix`. Quelle pourrait être cette machine ?

On a ici 2 machines identifiées comme étant des Windows : 

- `172.20.10.3` (`nofix`)

- `172.20.10.2` qu'on ne connaît pas !

On a ici une adresse IP qui pourrait correspondre à celle de notre attaquant. Retournons sur [Wireshark](https://www.wireshark.org/download.html) afin de voir quelles sont les informations échangées entre ces deux machines.

![Ce qu'envoi la machine suspecte à celle de `nofix`.](/img/write-up/nbctf2024/conversation_c2.png)

<figcaption>Ce qu'envoi la machine suspecte à celle de `nofix`.</figcaption>

Afin de suivre la **conversation** (ce que se "disent" les machines entre-elles), on fait clic droit sur un paquet > Suivre > Flux TCP.

![Conversation entre les deux machines.](/img/write-up/nbctf2024/flux.png)

<figcaption>Conversation entre les deux machines.</figcaption>

#### Spyware

> Contraction de *"spy"* (espionner) et *"software"* (logiciel) : un logiciel espion.

Les personnes parmi vous qui ont déjà tapés quelques commandes sur le terminal Windows ont déjà compris : 

- `type` permet d'afficher le contenu d'un fichier (`cat` sous Linux).

- `dir` permet de lister ce qui se trouve dans un dossier (`ls` sous Linux).

L'attaquant (bleu) lance des commandes pour voir le contenu de dossiers et afficher les fichiers à l'intérieur. La victime (rouge) exécute les commandes et envoi cela à l'attaquant. **L'attaquant est en train de récupérer ce que contient l'ordinateur de `nofix`.**

En fait, le malware permet un accès à distance à la machine de `nofix`. De son côté, l'attaquant doit attendre que `nofix` exécute son malware. Se faisant, la machine victime envoi une demande de connexion, que l'attaquant accepte. Tant que le malware s'exécute, l'attaquant peut exécuter des commandes sur la machine infectée.

On sait quelles commandes ont été exécutées par l'attaquant, il ne vous reste qu'à écrire le flag.

L'attaquant aurait aussi pu lancer un programme, un script Powershell... Tout ce qu'on aurait pu faire depuis un invite de commande !

### Pour conclure et aller plus loin

Ce *spyware* est très basique. En réalité, il initie une connexion TCP avec l'adresse IP de l'attaquant (en dur dans le code), une fois cette connexion établie, il exécute les commandes que lui envoie l'attaquant. Enfin, le programme retourne le résultat des commandes à l'attaquant.

Il n'y a pas de chiffrement de la connexion comme cela est souvent le cas pour un vrai malware. Cela rend la compréhension des actions de l'attaquant plus difficile. Il n'y a pas non plus de mécanisme d'obfuscation pour que le programme se cache d'éventuels antivirus. En effet, pour faciliter le challenge et le développement du malware, j'ai désactivé toutes les protections présentes sur Windows.

Sur une machine Windows classique, ce programme devrait être bloqué par l'antivirus Windows Defender de base et par n'importe quel autre digne de ce nom. De plus, la connexion se faisait sur un port exotique, elle aurait dû être bloquée par le pare-feux Windows. Une bonne raison, s'il en fallait une de plus, de ne pas désactiver les solutions de sécurité.

J'espère que cette suite de challenge vous aura plu. Pour ma part, j'ai adoré la créer ainsi que d'écrire ces writeups !

### Références

- [Wireshark](https://www.wireshark.org/download.html)

- [NetworkMiner](TODO)
