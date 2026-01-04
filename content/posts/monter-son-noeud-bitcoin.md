---
title: "Monter son noeud Bitcoin pour le fun"
date: 2026-01-04T10:25:27+02:00
draft: false
---

## Résumé

Le réseau Bitcoin ne se résume pas aux mineurs et au prix du jeton. Il s'agit avant tout d'un réseau décentralisé pair-à-pair dont la technologie vaut le détour. Je détaille ici un projet de vacances qui a pour objectif de monter un noeud Bitcoin ainsi qu'une interface web permettant de visualiser concrètement l'activité du réseau. C'est l'occasion de parler infrastructure, développement et Bitcoin, sans aborder le volet financier. Monter un noeud est finalement assez simple (même pour un non-informaticien) et permet de soutenir le réseau et sa décentralisation. Je vais considérer que le lecteur dispose de notions basiques sur le fonctionnement du protocole Bitcoin (comme moi quand j'ai commencé ce projet). Par exemple, ne détaillerai pas le concept de preuve de travail ni de consensus.

J'ai également mis en place un site web qui expose certaines données de mon noeud afin de rendre cela plus concret : [https://bitcoin.tourbillonsoftware.fr/](https://bitcoin.tourbillonsoftware.fr/). 

<meta property="og:image" content="https://nathan-out.github.io/img/blog/monter-son-noeud-bitcoin-pour-le-fun/illustration.png">
<meta property="og:description" content="Je détaille ici un projet de vacances qui a pour objectif de monter un noeud Bitcoin ainsi qu'une interface web permettant de visualiser concrètement l'activité du réseau.">
<meta property="twitter:description" content="Je détaille ici un projet de vacances qui a pour objectif de monter un noeud Bitcoin ainsi qu'une interface web permettant de visualiser concrètement l'activité du réseau.">

![](/img/blog/monter-son-noeud-bitcoin-pour-le-fun/site.png)

<figcaption>L'interface web que j'ai montée à partir des données du noeud.</figcaption>

## Vue d'ensemble du réseau

D'un point de vue protocolaire, il existe 3 types d'acteurs :

- **full node** : valident tous les blocks, et par extensions les transactions, disposent d'une copie de la blockchain et relaient les informations au reste du réseau. C'est de ce type d'acteur dont il est question ici.

- **mining nodes** (les mineurs) : construisent les blocks et minent afin de valider un block et gagner ainsi la récompense. Il s'agit sans conteste des acteurs les plus connus du réseau.

- **clients nodes** (SPV) : valident les entêtes des blocks uniquement et ne détiennent pas une copie de la blockchain. 

### Full node

Le site officiel de Bitcoin détaille comment [faire tourner un full node](https://bitcoin.org/en/full-node). Un noeud complet va donc télécharger l'entièreté de la blockchain, depuis le bloc 0 (`genesis block`) jusqu'au bloc le plus récent et vérifier que les transactions respectent les règles du réseau (liste non exhaustive) :

- création de Bitcoin lors du minage d'un block

- vérification des signatures

- format des données dans les blocks

- vérification de la non [double-dépense](https://en.wikipedia.org/wiki/Double-spending) qui est un des problèmes fondamentaux adressé par Satoshi Nakamoto dans l'article fondateur de Bitcoin

Evidemment, si une transaction ou un block ne rempli pas les conditions, il est rejeté. En fait, un noeud complet permet de **vérifier soit-même la légitimité de la blockchain** : *don't trust, verify*. Détenir un full node permet aussi de **renforcer la confidentialité de ses transactions** car il n'expose pas certaines informations susceptibles de lier votre identité à un wallet. De fait, un mineur dispose d'un pouvoir réduit sur le réseau et ce sont les full nodes qui vérifient le travail des mineurs. La preuve de travail des mineurs les dissuadent d'essayer de manipuler le réseau car les full nodes pourraient ne pas valider leurs blocs. Ils auraient donc travaillé pour rien, engendrant un coût.

A noter que les règles du réseau sont transparentes et accessibles à tous. Un changement de ces règles provoquerait un `hard fork`, c'est-à-dire la création d'une nouvelle blockchain et d'un nouveau protocole. 

## Faire tourner un noeud complet

Puisqu'un noeud complet requiert de télécharger et vérifier l'entièreté de la blockchain, cela nécessite quelques **700GB** de stockage (début 2026). Disposer d'une capacité de stockage de cet ordre de grandeur en cloud impose quelques coûts (720Go NVMe chez Ionos, comptez 30€ HT/mois). Fort heureusement, les développeurs ont adressé ce problème en ajoutant une fonctionnalité de *pruning*. Cela permet de définir la taille de la blockchain que vous stockez. Un noeud qui stocke toute la blockchain est un *full node* tandis qu'un noeud implémentant le *pruning* est appellé *pruned node*. Néanmoins, dans les deux cas, **l'entièreté de la blockchain est vérifiée par le noeud**. 

Monter son propre noeud permet de ne dépendre d'aucun intermédiaire et d'être complètement souverain sur ses transactions, en plus de renforcer la décentralisation du réseau. Si vous souhaitez monter un noeud complet non *pruned* je vous recommande de le faire tourner depuis chez vous, et pas sur un VPS. A défaut, un *pruned node* est une bonne alternative ou un SPV, même si ce dernier ne fournit pas autant de garanties qu'un node classique. 

Je dispose d'un petit VPS dimensionné pour faire tourner quelques services. J'ai donc mis en place un pruning de 550MB (taille minimale autorisée). Il faudra également ouvrir certains ports en fonction des services que votre noeud rendra au réseau. Enfin, attention si vous êtes facturé à la volumétrie du trafic : **vous devez télécharger toute la blockchain et répondre aux autres noeuds du réseau** (ce dernier point n'est pas obligatoire et est configurable). A l'heure où j'écris ces lignes, j'estime à **6-7 jours** le temps de synchronisation de mon noeud. 

L'architecture du système est assez simple : 

- un reverse proxy nginx conteneurisé qui expose les ports accessibles depuis internet

- un [conteneur Docker Bitcoin Core](https://hub.docker.com/r/ruimarinho/bitcoin-core)

Le docker-compose :

```yml
  bitcoin-node:
    image: ruimarinho/bitcoin-core:latest
    container_name: bitcoin-node
    restart: unless-stopped
    ports:
      - "127.0.0.1:8332:8332" # JSON RPC/REST mainnet (conf sécurisée)
      - "8333:8333" # P2P mainnet
    volumes:
      - ./bitcoin-node/bitcoin-data:/bitcoin # Stockage de la blockchain
      - ./bitcoin-node/bitcoin.conf:/home/bitcoin/.bitcoin/bitcoin.conf # Configuration du noeud (notamment la conf pruned)
    networks:
      - web # Réseau du reverse-proxy lui aussi conteneurisé
```

Notez que le port `8332` doit être traité avec vigilance car il peut exposer des **fonctionnalités sensibles**. Pour plus de détails, je vous laisse voir les différentes documentations en ligne sur le sujet (dont certains liens sont disponibles plus haut). 

### Configuration du noeud

Le noeud expose une interface RPC (*Remote Procedure Call*) qui permet d'intéragir avec lui en appellant des fonctions spécifiques. Tous les endpoints RPC sont décrits dans [la documentation officielle](https://developer.bitcoin.org/reference/rpc/). Certains endpoints permettent de récupérer des informations sur le noeud, sur la blockchain, d'effectuer des actions sur le réseau, d'envoyer des transactions, d'intéragir avec des wallets, etc... C'est pourquoi il faut absolument sécuriser cette interface et **ne pas exposer le RPC à tout internet** (surtout si vous vous en servez pour vos propres transactions).

Bitcoin Core supporte un fichier de configuration, voici ma configuration minimale :

```conf
# Activer le RPC
server=1

# Authentification RPC
rpcuser=REDACTED
rpcpassword=REDACTED

# Lier le RPC seulement sur le réseau Docker (sécurisé). Réseau Docker web (voir docker-compose)
rpcbind=0.0.0.0
rpcallowip=172.18.0.0/16 # va dépendre dans quel réseau Docker se trouve le noeud

# Port RPC par défaut
rpcport=8332

# Pruned node pour limiter l’espace disque
prune=550

# Optionnel : activer le P2P pour aider le réseau
listen=1
```

**Note :** je reviens plus bas sur cette configuration pour optimiser la consommation RAM.

## Interface

Mon idée était de rendre tout cela plus concret en proposant une interface web accessible à tous. Le RPC n'était exposé qu'au réseau Docker, il faut donc ajouter un autre composant. C'est pourquoi j'ai codé une API en Python qui est le backend de l'interface web. 

![](/img/blog/monter-son-noeud-bitcoin-pour-le-fun/infra.png)

<figcaption>Schéma de l'infrastructure. Le RPC n'est pas exposé directement sur Internet, l'API Python n'expose que certains endpoints RPC inoffensifs.</figcaption>

Le site affiche donc l'état du noeud, à quel réseau il est connecté, à combien d'autres noeuds il est connecté, l'uptime... L'interface affiche également les 3 derniers blocks du réseau avec la plupart des informations contenues à l'intérieur.

Mon VPS (2 Go de RAM) ne semble pas assez puissant pour traiter autant de données ; environ 2 000 transactions par bloc pour lesquelles les données associées devraient être récupérées, traitées et sérialisées. Le reverse proxy et quelques autres services doivent également être pris en compte. J'ai essayé d'activer cette fonctionnalité sur la fonction RPC [getblock](https://developer.bitcoin.org/reference/rpc/getblock.html) via le paramètre `verbosity=2`. Cela semble avoir saturé le conteneur Docker et son volume, m'empêchant de redémarrer le conteneur car il utilisait toute la RAM et a ensuite été tué par l'hôte. J'aurais aimé afficher les données contenues dans les transactions afin de pouvoir explorer la blockchain en profondeur. En particulier, les messages laissés par les utilisateurs dans [OP_RETURN](https://developer.bitcoin.org/reference/rpc/getblock.html). 

## Faites votre propre interface

Voici les endpoints exposés par l'API Python, faites vous plaisir ! J'ai mis en place un rate-limit de 10 requêtes par secondes avec pic autorisé à 20.

```python
# BASE API : https://bitcoin.tourbillonsoftware.fr/api/

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/getblockchaininfo")
def getblockchaininfo():
    rpc = get_rpc()
    info = rpc.getblockchaininfo()
    return info

@app.get("/uptime")
def uptime():
    rpc = get_rpc()
    return rpc.uptime()

@app.get("/getnettotals")
def getnettotals():
    rpc = get_rpc()
    return rpc.getnettotals()

@app.get("/getconnectioncount")
def getconnectioncount():
    rpc = get_rpc()
    return rpc.getconnectioncount()
```

## Optimisation

A toute fin utile, voici une amélioration du fichier de configuration `bitcoin.conf` qui vise à réduire la consommation de RAM. Notez que certains paramètres peuvent changer le comportement de votre noeud (RTFM). Si vous voulez creuser, voici deux sites qui m'ont été utiles :

- [Bitcoin Stack Exchange - How to run node bitcoind in a low memory environment](https://nathan-out.github.io/)

- [Github Bitcoin Doc - Reduce Memory](https://github.com/bitcoin/bitcoin/blob/master/doc/reduce-memory.md)