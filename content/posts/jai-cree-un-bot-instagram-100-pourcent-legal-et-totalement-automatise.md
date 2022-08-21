---
title: "J'ai créé un bot Instagram 100% légal et totalement automatisé"
date: 2022-08-05T21:30:44+02:00
draft: false
---
Instagram regorge de bots (ou robots), la plateforme leur mène une guerre totale avec des moyens colossaux et des technologies d'analyse du comportement poussées pour déterminer si un compte est managé par un robot ou non. Pourquoi ? Ces robots sont parfois mal intentionnés : arnaque, harcèlement, désinformation ou tout simplement spam. C'est pourquoi si un robot se fait détecter, il est automatiquement supprimé de la plateforme, maintenir un réseau de bot est donc compliqué et risqué, on peut tout perdre très rapidement. De l'autre côté, les créateurs de bots rivalisent d'ingéniosité afin de rester invisible, c'est un sujet passionnant, une vraie guerre invisible, mais ça n'est pas le sujet d'aujourd'hui. 

Il est existe pourtant une technique afin de créer un bot légitime aux yeux d'Instagram : utiliser l'API de Meta (ex Facebook). Ce bot est donc connu de la plateforme, il possède quelques limitations mais fera parfaitement l'affaire pour ce que souhaite en faire.

## Un bot, mais pour quoi faire ?

Il y a quelques temps, je suis tombé sur cet article au titre plutôt sexy : [How I Eat For Free in NYC Using Python, Automation, Artificial Intelligence, and Instagram](https://medium.com/@chrisbuetti/how-i-eat-for-free-in-nyc-using-python-automation-artificial-intelligence-and-instagram-a5ed8a1e2a10). API, Intelligence Artificielle, Python, automatisation et analyse de données : ça vend du rêve ! Pour résumer l'article, un ingénieur, avec beaucoup trop de temps à perdre visiblement, s'est réveillé un matin avec une envie : manger gratuitement dans les restaurants de New-York. Puisqu'il a des compétences en informatique, il décide de créer un bot Instagram qui va analyser les publications culinaires qui fonctionnent le mieux à New-York, de les re-poster sur son compte, d'observer ce qu'il se passe et, bien sûr, de monétiser sa visibilité. Tout cela, boosté avec de l'intelligence artificielle et des techniques de big data pour être le plus efficace possible. L'objectif final est d'échanger la visibilité de son compte contre des repas gratuits dans les meilleurs restaurant de la ville, du génie ! Benjamin Code en a fait [une vidéo](https://www.youtube.com/watch?v=rKK3z1H3vYs) qui récapitule le projet, je vous invite aussi à lire l'article, c'est passionnant.

Je me suis dis que je pouvais moi aussi tenter l'expérience, à plus petite échelle et avec moins de connaissances que lui. L'objectif est de créer un compte qui pourrait publier du contenu automatiquement, sans analyse de donnée (type big data) et sans IA, dans un premier temps.

## A la recherche de contenu

Publier du contenu c'est bien, mais encore faut-il en avoir. Ca tombe bien puisque quelques jours auparavant, un ami m'a transmis [ce repo Github qui regroupe des API publiques](https://github.com/public-apis/public-apis). C'est une vraie mine d'or et l'une de ces API m'a fait de l'oeil : [Metropolitan Museum of Art](https://metmuseum.github.io/). Elle regroupe une quantité colossale d'oeuvres d'art (avec des photos !). Le principe est simple : j'envoie une requête à l'API, elle me répond avec une oeuvre d'art et des informations en plus, voilà pour le contenu.

## L'API d'Instagram : voyage en terre inconnue

Comme expliqué plus haut, pour avoir un bot "legit" je dois passer par l'API d'Instagram. Le truc qui m'a fait perdre beaucoup, **beaucoup**, de temps a été de comprendre comment avoir un accès. En effet, cette API nécessite une configuration particulière assez pénible et laborieuse à mettre en place :
- un compte Facebook Developer (un numéro de téléphone valide vous sera demandé)
- une page Facebook liée à ce compte developer
- un compte Instagram professionnel (impossible de le lier à l'API sinon)
- une tripotée de configuration, d'identifiants et de secrets à avoir
- remuez le tout, avec quelques tutos en ligne et un peu de temps, vous devriez arriver à faire votre première requête

A noter quand même qu'Instagram ou Facebook ne vous laissera pas tranquille. En effet, il m'est arrivé de jongler entre mes comptes persos et ceux pour mon bot... ce qui a eu pour effet de bloquer mon compte bot ! Tout ça parce que j'avais une activité "suspecte" avec un compte nouvellement créé, quand je vous disais que ces grandes plateformes faisaient la guerre aux bots... Il faut parfois attendre un peu, parfois valider son mail, bref c'est pas super pratique.

Une fois cela fait, il "suffit" de comprendre l'ordre des requêtes pour arriver à poster une image sur Instagram. Là encore ça n'est pas si simple mais une fois qu'on a compris l'ordre et ce que nous retourne chaque requête, le script est plutôt facile à coder. Nous voilà donc avec un accès et un compté légitime et des données en pagaille. Ne reste plus qu'à coder le robot pour qu'il aille de lui-même chercher les images dont il a besoin et les publier.

## He's alive !

Cette partie a été la plus longue, de loin. J'ai passé quelques soirées et raccourci des nuits mais c'était la partie la plus fun, bien plus que d'obtenir des accès à l'API. Il ne reste qu'à traiter les données pour les publier.

### Du tris

Le premier problème est que tous les éléments de l'API ne contiennent pas d'image. Par exemple lorsqu'on requête l'API pour obtenir un élément de sa collection, il arrive qu'il n'y ait pas d'images : on a un titre un auteur et tout un tas d'autres données... mais pas d'images. Premier filtre sur les éléments qui ont une image. 

Ensuite, certains des éléments de l'API ont plusieurs images, une image "primaire" et des images "secondaires", qui sont des photos de l'oeuvre prises sous différents angles. Second filtre pour avoir des éléments avec plusieurs images.

Instagram n'accepte pas tous les ratios d'images ; une image trop étirée verticalement ou horizontalement ne sera pas acceptée pour être publiée. Troisième filtre.

Enfin, en faisant mes tests j'ai remarqué que certaines catégories d'oeuvres retournaient des images vraiment très peu intéressantes : des fragments de poterie, par exemple. Quatrième filtre.

Une fois qu'on a des images à peu près jolies et publiables sur Insta, on peut continuer à perfectionner notre publication. J'aime bien *rendre à César ce qui est à César* ; c'est pourquoi j'ai eu à coeur de mettre toutes les infos en ma possession (quand j'en avais) pour chaque oeuvre publiée :
- titre
- auteur (+ dates de naissance et de mort)
- type d'oeuvre
- collection
- crédits
- et d'autres...

Reste une chose **primordiale**, les **HASHTAGS**.

### Un peu d'intelligence...

Ces foutus hashtags m'en auront fait voir de toutes les couleurs ! Pour le faire courte, c'est ce qui va permettre de référencer ma publication, il faut donc qu'ils soient :
- précis
- relativement nombreux (mais pas trop non plus)
- cohérents avec la publication

Pour cela j'ai découpé mes hashtags en 3 catégories :
- les hashtags statiques, c'est-à-dire ceux qui seront toujours là, par exemple "#art", "#masterpiece", "#picture", "#museum", "#exposition".
- les hashtags dynamiques, ceux directement issus des données de l'oeuvre sur la publication : nom de l'oeuvre, de l'artiste, type, date, culture...
- les hastags **intelligents**, qui ne sont ni issus des données de l'image, ni de ma main. Ils doivent être **tirés** de l'image.

J'avais dans l'idée d'extraire des mots-clefs de l'oeuvre, par exemple pour l'image ci-dessous on aurait pu avoir : *bridge, phone, sunset, hand, road, sky...*

![Image d'exemple](/img/blog/archie-the-bot/Image_created_with_a_mobile_phone.png)

*"Quoi de mieux que **l'intelligence artificielle** pour faire ça ?"* - me demandais-je naïvement. 

Effectivement il existe des modèles d'IA pour effectuer ce travail, je l'ai ajouté à mon projet, j'ai trouvé un peu de code sur internet pour le faire en python (le langage que j'utilise), et ça fonctionne ! Par contre les résultats sont très, **très** mitigés. Par exemple, pour l'image du dessus, le modèle trouvait un "ours polaire", une "montgolfière" et autres bizareries...

### ...ou pas !

Alors j'ai fais des hypothèses, j'ai cherché, j'ai galéré, j'ai même pensé que je faisais "rentrer" mon image dans le mauvais sens dans mon IA... mais non ! Le modèle est juste mauvais. Pour comprendre il faut que j'explique [l'apprentissage supervisé](https://fr.wikipedia.org/wiki/Apprentissage_supervis%C3%A9) en deux mots.

On donne une immeeeeennnnse quantité d'images au modèle qu'on souhaite entraîner avec à chaque fois des mots-clefs associés à chaque image (par exemple une image de canard avec comme mot-clef "canard", "oiseau", etc...). Pour chaque image l'IA va tenter d'elle-même de trouver des mots-clefs et une fois ses résultats donnés, elle les compare avec ce qu'elle aurait dû trouver ("canard", "oiseau", etc...), et elle "s'améliore" en fonction de l'écart entre ce qu'elle a prédit et ce qu'elle aurait dû trouver.

Le soucis de cette méthode c'est que l'IA ne va reconnaître que ce sur quoi elle s'est entraînée. Plus on souhaite une IA performante sur une large gamme d'image, plus on doit lui donner une quantité de donnée, un *dataset*, grand et bien construit. Si notre IA doit seulement différencier un panneau stop d'un passage piéton, c'est "facile". Si, en revanche, elle doit pouvoir détecter *n'importe quoi* sur *n'importe quelle image*, c'est une autre paire de manches.

Le soucis c'est que le modèle pré-entraîné que j'ai voulu utilisé était trop généraliste, il pouvait tout reconnaître mais faisait beaucoup trop d'erreurs (je n'avais parfois aucun mot en rapport, même de loin, avec mon image !). Tout ça pour dire qu'à moins d'entraîner moi-même l'IA sur un dataset d'oeuvre d'art proches de celles sur lesquelles je souhaitais extraire les mots-clefs, je ne pourrais rien en tirer. Je ne dispose ni du temps, ni de la puissance de calcul, ni de l'espace de stockage, ni des compétences, ni du dataset pour le faire. Une prochaine fois peut-être ?

En vérité ce sujet mériterait un article à part entière tellement c'est bien plus compliqué et passionnant que ça.

## Encore un petit effort

Après ce détour dans les méandres de l'IA, me voilà de retour sur mon bot, sans mes mots-clef "intelligents", tant pis ! Après quelques tests, quelques sesssions de "debug", j'ai un programme fonctionnel ! Il est capable de rechercher, traiter et poster une publication Instagram ! Ne reste qu'à le mettre sur un serveur, programmer ses horaires de diffusion et surveiller que tout se passe bien dans un premier temps, avant de le laisser vivre sa vie de bot Instagram. Il publie tous les jours à **10h et 20h**.

Sans plus attendre, voilà Archie, le robot ! Je vous invite à aller voir son compte [@archie_the_aesthete](https://www.instagram.com/archie_the_aesthete/) pour ne rater aucune publi, pensez à vous abonner et à lâcher un like, ça lui fera plaisir ! :).

![Une image du compte Instagram @archie_the_aesthete.](/img/blog/archie-the-bot/archie_compte.png)

Bon, pour le moment le succès n'est pas au rendez-vous, mais ça viendra...

Quelques-unes de ses publications :

![Une publication du compte Instagram @archie_the_aesthete.](/img/blog/archie-the-bot/archie1.png)

![Une publication du compte Instagram @archie_the_aesthete.](/img/blog/archie-the-bot/archie2.png)

![Une publication du compte Instagram @archie_the_aesthete.](/img/blog/archie-the-bot/archie3.png)

## La suite ?

Il y aurait encore des tas et des tas d'améliorations à apporter. On est encore loin de [l'article qui m'a inspiré ce projet](https://medium.com/@chrisbuetti/how-i-eat-for-free-in-nyc-using-python-automation-artificial-intelligence-and-instagram-a5ed8a1e2a10), il manque l'IA bien sûr et le côté analyse de données. Cela fait 3 semaines maintenant qu'Archie tourne et je ne vois pas vraiment d'évolution du compte, il n'y a pas d'abonnés ni de tendance à la hausse. Pour monayer ce compte, on repassera mais le projet était vraiment cool j'ai appris énormément de choses ! Peut-être que je rendrai le code libre, pensez à checker [mon github](https://github.com/nathan-out).

Je pense à une V2 avec plein de nouvelles fonctionnalités qui permettraient d'améliorer la visibilité du robot :
- création d'une story à chaque nouvelle publication
- des oeuvres plus alléchantes (certaines ne sont pas du tout "instagrammables", faut être honnête)
- garder un feed cohérent (des couleurs proches les unes des autres entre les publications), c'est ce qui fait "pro", il parait
- de l'analyse de données pour savoir ce qui fonctionne, à quels horaires publier, etc...

Bref il reste encore du boulot et des tas de trucs intéressants à explorer ! Sur ce, bon vent, Archie !

![Une image de la BD rétro "Archie the robot"](/img/blog/archie-the-bot/archie_the_robot.png)
