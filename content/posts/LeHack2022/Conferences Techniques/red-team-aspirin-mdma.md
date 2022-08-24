---
title: "Swapping aspirin formulas with MDMA while red teaming a billion dollar pharmaceutical"
date: 2022-06-30T13:56:22+02:00
draft: false
---
*Aman Sachdev*

**<u>Objectif :</u>** accéder au SDMS (scientist data managment system).

# 1. Internet to intranet

L'équipe a commencé par essayer d'infiltrer la compagnie par le biais de ses applications webs. Ils ont trouvé une RCE, une injection SQL, une faille sur l'authentification SMTP, des défauts dans la configuration SAP ainsi que des mots de passe par défaut sur certaines installations. Ils ont également effectué de l'OSINT sur des employés via des fuites de bases de données, cela leur a permis de disposer de mots de passe. Cependant, les accès qu'ils ont pu avoir étaient limités par rapport à l'objectif final.

Ils ont donc décidé d'une approche physique. En rôdant dans une voiture à proximité du campus, ils sont parvenus à capter une borne wifi de l'entreprise et à s'infiltrer dans le réseau interne par ce biais.

# 2. Eviter le SOC et exfiltrer de la donnée

Puisque c'est une grande entreprise, il y a une multitude de pare-feux, de bâtiments, de réseaux... Pour bien cartographier, ils ont fait des scans, testé les VLAN, écouté le réseau et fait du phishing interne. Ils ont utilisé des exploits pour les pare-feux ce qui leur a donné accès à une base de données chiffrée.

Les principales failles sont des logiciels installés par des tiers, qui ne sont plus maintenus/utilisés. Pour les exploiter, les auditeurs ont lu la documentation et identifié les mots de passe par défaut. Une faille importante était une base de données lancée avec les privilèges super-utilisateurs. Ces multiples failles leurs ont permis d'avoir accès aux notes des scientifiques ainsi qu'à un logiciel qu'ils utilisent pour avoir accès au SDMS. Ils ont posé une backdoor type Man-in-the-browser, lorsqu'une personne s'est connectée au SDMS, ils ont pu récupérer ses identifiants pour avoir un accès. Jackpot ! le SDMS contient les formules chimiques des molécules utilisées par le laboratoire et qui font l'objet de recherches très pointues et dont l'investissement est de plusieurs milliards de dollars. Les auditeurs étaient donc en mesure de **modifier, supprimer, intervertir, exfiltrer les formules et les notes des scientifiques.**

L'objectif était atteint mais les auditeurs ont voulu poursuivre l'exploration.

# 3. Accès à la ligne de production

Grâce à l'exploitation d'une configuration par défaut, les auditeurs ont été en mesure d'accéder à une machine vulnérable à une 0-day. C'était gagné : ils étaient en mesure de contrôler des machines de la ligne de production, création de boîtes, empaquetage...

L'objectif d'accéder au SDMS était atteint, avec en plus un bonus : les accès à la ligne de production.

# Les faiblesses identifiées & résumé

- trop de scans possibles : trop de données en libre accès (OSINT sur les employés par exemple), mots-de-passes inchangés après des fuites... la reconnaissance a été trop facile.
- services mal configurés, ou configurés par défaut.
- exploit possible sur des applications exposées sur le web.
- création de binaires signés, obfuscation des payloads, utilisation de token impersonation et désactivation des pare-feux.
- utilisation de [Cobalt Strike](https://www.cobaltstrike.com/), de VPN, de Meterpreter et de [ABPTTS](https://github.com/nccgroup/ABPTTS).
- SSL impersonation, One Drive/DropBox -> difficile de bloquer les bons flux avec ces outils là sur le réseau. Cela a facilité la discrétion des auditeurs.
- exfiltration et stéganographie grâce à [CloakifyFactory](https://github.com/TryCatchHCF/Cloakify), bind shell over HTTPS, DNS tunneling, ICMP, [Empire Dropbox](https://github.com/EmpireProject)...
