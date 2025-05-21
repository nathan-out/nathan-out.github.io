---
title: "Shellcode en poupée russe"
date: 2025-05-21T10:25:27+02:00
draft: false
---

## Résumé

Le principe de fonctionnement de ce code est similaire à celui des poupées russes. Le code contient quelques instructions en claires, qui permettent de déchiffrer les instructions suivantes puis s'appelle lui-même, et ainsi de suite jusqu'au code final qui contient le flag. Pour chaque étape, il faut comprendre la façon dont sont chiffrées les instructions suivantes, puis les importer de nouveau dans IDA pour les analyser. J'ai identifié 6 étapes (*stage*) différentes et j'en détaille quelques-unes.

La première étape consiste à charger les instructions sur la pile puis à l'exécuter. Les étapes suivantes suivent toutes le même principe :
1. récupération de l'adresse de l'instruction courante
2. enregistrement de l'adresse de début des instructions chiffrées
3. enregistrement du nombre d'instructions à déchiffrer
4. si le déchiffrement est terminé, alors on *jump* sur les instructions déchiffrées, sinon on continue le déchiffrement

*Binaire `such_evil` tiré du [FLARE On Challenge](https://flare-on.com/) 1ère édition (2014) - challenge 3*

## Première ouverture : comprendre

Dire que j'étais perdu lors de la première ouverture du binaire dans IDA est un euphémisme. Je m'attendais à avoir directement du code à *reverse*. En réalité, il ne s'agit pas du code qui nous intéresse mais plutôt d'une sorte de pré-code qui sert à appeler la fonction principale, ici `sub_401000`. Je ne rentrerai pas dans le détail de ce que fait ce code, ChatGPT sait très bien l'expliquer.

![Ce que montre IDA lors de la première ouverture du binaire. Ce qui nous intéresse se trouve à partir de la fonction `sub_401000`.](/img/blog/flareon/3-start_IDA.png)

<figcaption>Ce que montre IDA lors de la première ouverture du binaire. Ce qui nous intéresse se trouve à partir de la fonction `sub_401000`.</figcaption>

## Stage 1 : chargement du code et exécution

La première fois que j'ai ouvert `sub_401000`, je n'ai rien compris. Je n'avais jamais vu un code aussi étrange. On voit des déclarations d'énormément de variables locales, du code qui manipule la pile (normal lors d'un appel à une fonction), puis ce pattern étrange qui consiste à charger dans `eax` puis de le pousser sur la pile, pour terminer avec un `call eax` :

```asm
sub_401000 proc near

# déclarations de beaaaauuuucoup de variables locales
var_201= byte pr -201h
var_200= dword ptr -200h
var_1FC= byte ptr -1FCh
...
var_1= byte ptr -1

[code usuel de début de fonction du type push ebp etc...]

# début du pattern étrange
mov     eax, <1 octet>
mov     [ebp+var_N], al
mov     eax, <1 octet>
mov     [ebp+var_N+1], al
...
lea     eax, [ebp+var]
call    eax
sub_401000 endp
# fin fonction
```

Le principe de ce code est d'allouer sur la pile un espace assez grand pour acceuillir le code, de le pousser octet par octet, puis de l'exécuter, comme le montre le schéma ci-dessous. **On souhaite donc ici récupérer le code poussé sur la pile.** Il s'agit de la seule étape pour laquelle j'ai utilisé une technique d'analyse dynamique. En effet, il aurait été fastidieux de devoir reconstruire à la main l'ensemble des intructions exécutées à l'étape suivante, d'autant qu'elles ne sont pas chargées dans l'ordre. L'idée est donc de récupérer les instructions une fois chargées afin de les analyser.

![Remplissage de la pile avec les instructions contenues comme variables locales de la fonction `sub_401000`](/img/blog/flareon/1-chargement_shellcode_stage1.png)

<figcaption>Remplissage de la pile avec les instructions contenues comme variables locales de la fonction `sub_401000`. A la fin du code, on charge l'adresse du début de la pile et le programme *jump* dessus, ce qui exécute le code.</figcaption>

Pour les connaisseurs, le fait d'exécuter des instructions sur la *stack* peu déjà être un élément signant ; ce n'est pas un comportement souhaitable d'un point de vue sécurité. D'ailleurs, des protections sont normalement mises à la compilation pour empêcher cette possibilité lors de l'exécution. Dans ce challenge, les protections sont désactivées, comme le montre la valeur de l'entête `DLLCharacteristics` à 0. L'outil [PE-bear](https://github.com/hasherezade/pe-bear) permet d'obtenir cette information (et bien d'autres).

Pour ce faire on pose un *breakpoint* sur l'instruction `call eax` et on lance l'exécution. On obtient l'image ci-dessous.

![](/img/blog/flareon/2-stage1_breakpoint.png)

<figcaption>Breakpoint sur IDA, l'exécution est arrêtée juste avant l'exécution du shellcode.</figcaption>

En double-cliquant sur `eax`, IDA nous emmène au shellcode qui est complet à ce stade de l'exécution. IDA inteprête ces données sur la *stack* comme du binaire par défaut, mais on est à peu près sûr qu'il s'agit en partie d'instructions (sinon pourquoi *call* dessus ensuite ?). On peut forcer l'inteprétation du binaire en code avec la touche "C". On observe alors des instructions cohérentes et d'autres plus exotiques comme `or` ou `adc`. Même si ces dernières sont totalement valides, elles peuvent indiquer que les instructions sont chiffrées car ce sont des instructions rares.

Dans la capture ci-dessous, j'ai déjà commenté le code et renommé certains éléments pour faciliter la compréhension. A partir de l'offset `0x0019FD50`, il s'agit de données qui sont en fait encodées. J'ai surligné les instructions exotiques en jaune.

![Contenu de la pile lors de l'exécution du programme. On observe des patterns d'instructions plutôt cohérents en haut et des instructions plus rares en bas.](/img/blog/flareon/3-fin_stage1.png)

<figcaption>Contenu de la pile lors de l'exécution du programme. On observe des patterns d'instructions plutôt cohérents en haut et des instructions plus rares en bas.</figcaption>

Un lecteur attentif pourrait se dire que `call $+5` est aussi une instruction exotique, et il aurait raison. Néanmoins, il s'agit en fait d'une technique plutôt maligne pour exécuter la suite du shellcode, j'en reparle dans la partie 2. On arrive à la fin de l'étape 1, il est temps de plonger plus profondément.

### Pour résumer le stage 1

Bien comprendre cette première partie est primodiale. Tout d'abord, on ignore le "pré-code" pour se concentrer sur la fonction principale qui est `sub_401000`. Le code dans cette fonction est inhabituel pour plusieurs raisons : 

1. déclarations de nombreuses variables locales

2. envoi, octet par octet, de données sur la pile

3. chargement de l'adresse de la pile puis *call* dessus

Pour analyser les données que contiennent la pile, on place un *breakpoint* sur l'instruction `call eax`, puis on exécute le programme dans IDA. De cette manière, on est en capacité de lire les données contenues dans la pile. On peut forcer IDA à interprêter ces données comme des instructions, ce qui nous donne du code en partie cohérent. Cela confirme l'hypothèse que le programme tente de nous cacher son véritable comportement. Néanmoins, du fait que certaines instructions sont exotiques, on émet l'hypothèse qu'elles sont chiffrées ou obscurcies. Il s'agit d'un procédé courant de programme malveillant. Dans les prochaines parties, je vais analyser ce code ainsi obtenu pour déterminer son véritable comportement.

## Stage 2 : grosse poupée

Reprenons la dernière image de la partie précédente en inteprêtant uniquement les instructions valides (non chiffrées). Note : j'ai renommé les étiquettes comme `xor_loop` et `stage2` pour faciliter la compréhension, elles sont nommées avec un nom bien moins signifiant dans le binaire.

![](/img/blog/flareon/4-stage2_commente.png)

<figcaption></figcaption>

Commençons par le début : `call $+5`. Le `$` signifie "adresse de l'instruction courante", et le `+5` "5 octets plus loin" ce qui équivaut à l'instruction suivante. **Cette instruction sert à exécuter l'instruction suivante, tout simplement**. En cherchant sur Internet, il s'agit d'une manipulation assez courante de code malveillant. La seconde instruction sert à stocker cette adresse (`0x0019FD34`).

S'ensuit un calcul d'adresse et d'offset pour que la boucle de déchiffrement (`xor_loop`) puisse charger l'adresse de l'instruction, la déchiffre et s'arrête une fois toutes les données déchiffrées. Ensuite, le programme peut *jump* sur `rebond_donnees_encodees` qui *jump* sur `stage2` qui n'est pas visible sur la capture ci-dessus. En effet, du code mort se glisse entre `rebond_donnees_encodees` et `stage2`. L'exécution reprend ensuite son court pour la troisième étape.

Arrêtons-nous sur le calcul d'adresse et d'offset. `mov esi, [esp]` permet de placer dans le registre `esi` la valeur qui correspond à l'adresse de l'instruction précédente, soit `0x0019FD34`. En effet, cette valeur se trouve dans `[esp]` du fait du `call $+5`. Ensuite, on ajoute à `0x0019FD34` `0x1C` ce qui donne `0x0019FD50`. Si on considère que `0x0019FD50` comme étant une adresse, on s'apperçoit que cela tombe pile après le `jmp near ptr stage2`. C'est comme si cette valeur était l'adresse de la première instruction à déchiffrer...

Regardons `mov ecx, 1DFh`. On peut émettre l'hypothèse que cette valeur représente le nombre d'instructions à déchiffrer car on voit un peu plus loin une comparaison d'`ecx` avec 0 ainsi qu'une décrémentation d'`ecx`. C'est typique d'une boucle avec compteur. Cela implique que la fin des données encodées se situent `0x1DF` instructions plus loin que la première, soit à l'adresse `0x0019FF2F`. Cela semble cohérent car cette adresse existe dans le programme. Enfin, les données sont déchiffrées avec un `xor byte ptr [esi], 66h`, puis la boucle passe à la donnée suivante avec `inc esi`. 

Pour illustrer tout ça, j'ai fait un schéma des différentes étapes. Il n'est pas 100% exact mais il illustre la logique de déchiffrement en 3 étapes.

![Illustration du déchiffrement du shellcode *via* un calcul d'adresse et d'offset. 1-l'adresse de début des données est calculée et le compteur est initialisé. 2-les données sont déchiffrées dans la boucle. 3-le compteur arrive à son terme ce qui dirige le flow d'exécution vers le code déchiffré.](/img/blog/flareon/5-desobfuscation_shellcode.png)

<figcaption>Illustration du déchiffrement du shellcode *via* un calcul d'adresse et d'offset. 1-l'adresse de début des données est calculée et le compteur est initialisé. 2-les données sont déchiffrées dans la boucle. 3-le compteur arrive à son terme ce qui dirige le flot d'exécution vers le code déchiffré.</figcaption>

Donc, pour déchiffrer les données et passer à l'étape suivante, il faut que nous fassions la même chose. **A savoir récupérer les octets entre les adresses `0x0019FD50` et `0x0019FF2F` pour les "xorer" avec `0x66`.** A ce moment là, peut-être que nous serons en face d'un code complètement lisible ! Pour cela, on sélectionne les données sur IDA puis dans le menu Edit > Export Data, on coche "raw bytes" pour que le fichier contiennent les données binaires. Un petit script Python suffi à décoder tout ça : 

```python
decoded = []
with open('shellcode_to_decode_xor_0x66.bin', 'rb') as f:
    d = list(f.read())
    for o in d: decoded.append(o^0x66)
    decoded = bytes(decoded)
with open('stage2.bin', "wb") as f: f.write(decoded)
```

## Stage X : avance rapide

On arrive au pourquoi du nom de cet article ; le challenge se répète de manière similaire **5 fois**. Le principe reste le même ; récupération de l'adresse de l'instruction, application d'un offset pour obtenir l'adresse des données chiffrées, initialisation du compteur, déchiffrement, *jump*, étape suivante.

![Schéma global du challenge ; il faut déchiffrer chaque étape pour arriver au flag.](/img/blog/flareon/6-poupees_russes.png)

<figcaption>Schéma global du challenge ; il faut déchiffrer chaque étape pour arriver au flag.</figcaption>

Ci-dessous des captures d'écran du code des étapes suivantes. Le code sera un peu différent si la clef de déchiffrement est une chaîne de caractère ou plusieurs octets, comme on peut le voir ci-dessous.

![Stage 2](/img/blog/flareon/7-stage2.png)

<figcaption>Stage 2</figcaption>

![Stage 3](/img/blog/flareon/7-stage3.png)

<figcaption>Stage 3</figcaption>

![Stage 4](/img/blog/flareon/7-stage4.png)

<figcaption>Stage 4</figcaption>

![Stage 5. Le flag se trouve tout en haut de la capture.](/img/blog/flareon/7-stage5.png)

<figcaption>Stage 5. Le flag se trouve tout en haut de la capture.</figcaption>

Et enfin, le flag ! Le format de flag du FLAREON est une adresse mail en flare-on.com. On notera l'humour des chall makers. Cependant, il semble qu'il y ait encore une étape...

## Une dernière poupée pour la route

Posons l'hypothèse que le programme est un véritable malware (spoiler : ça n'en est pas un). Nous sommes arrivés au bout de son obfuscation, c'est maintenant que la partie intéressante commence ! Je ne suis pas sûr de ce que le programme cherche à faire ici. Je pense qu'il cherche, de manière un peu détourné, une DLL qui contient un '3' dans son nom (probablement `kernel32.dll`) Ensuite, le programme cherche dans cette DLL un nom de fonction qui contient soit la chaîne `Fatal`, soit `Exit` (les chaînes sont écrites à l'envers sur la capture à cause de l'*endianness*).

Enfin, si une des deux chaînes est trouvée, le programme appelle la fonction avec en argument la chaîne `BrokenByte` suivi de l'octet `0x01`, ainsi que la valeur `0`. Etant donné la signature de la fonction, je pense que le programme cherche ici à appeller [FatalAppExitA](https://learn.microsoft.com/fr-fr/windows/win32/api/errhandlingapi/nf-errhandlingapi-fatalappexita).

![La dernière étape après l'obtention du flag.](/img/blog/flareon/7-stage6.png)

<figcaption>La dernière étape après l'obtention du flag.</figcaption>

On est maintenant certain que ce programme est inofensif. Si on l'exécute dans une VM, voici ce qu'on obtient.

![Résultat de l'exécution du programme dans une VM.](/img/blog/flareon/8-fin.png)

<figcaption>Résultat de l'exécution du programme dans une VM.</figcaption>
