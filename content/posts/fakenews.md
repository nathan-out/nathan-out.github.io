---
title: "Voir l'influence et détecter la manipulation de l'information"
date: 2023-01-08T18:10:23+01:00
draft: false
---

*Article de blog écrit à la suite du [Tournois de Renseignement et d'Analyse de CentraleSupélec (TRACS)](https://tracs.viarezo.fr/) édition 2022. En quelques mots, ce tournois a été l'occasion de concourir sur des thématiques diverses comme la data science, la cybersécurité ou encore la cryptanalyse.*

*Cet article reprend le challenge "Fakenews" de cette compétition et apporte la solution (writeup) que mon équipe et moi-même avons élaborés. Il se base sur un scénario fictif détaillé ci-dessous.*

## Le contexte et le challenge

Le sujet du challenge pose le ton :

> Sur TikBook, le principal réseau social du pays géré par la très puissante Eve Hilboss, circulent de nombreux éléments discréditant Paul Hityck. SOS (Special Operations Services), les services spécieux Indiumlandais, soupçonnent les liens entre Eve Hilboss et les services secrets eviliens. Ils demandent un appui pour soutenir leurs opérations sur ce dossier.

Pour simplifier la trame scénaristique, nous devons trouver **le pseudo qui est à l'origine d'une "campagne de désinformation" qui touche Paul Hityck**.

Par abus de langage, et pour simplifier la lecture, on va dire que les données sont issues de Twitter. En effet, les données du challenge indiquent un réseau social très proche de l'oiseau bleu. Je vais donc parler de *tweet* (contenu posté sur le réseau par un compte) et de *RT* (re-tweet ; lorsqu'un utilisateur partage le message d'un autre, en y ajoutant ou non du contenu), qui font partie du vocabulaire utilisé pour Twitter.

{{< alert >}}
Notez bien que les données fournies ici sont fictives, elles ne sont en aucun cas tirées d'un quelconque réseau social. Elles ont été générées pour le challenge.
{{< /alert >}}

Avec le contexte vient le fichier sur lequel nous allons travailler. Il est conséquent ; plus de 60Mo de texte. Il contient tous les tweets publiés, et leurs informations associées, sur une période de temps. Voici un extrait du fichier :

{{< highlight txt >}}

,Unnamed: 0,index,text,username,date
4458322,6879731,3283915,Money aint nothing if u know how to make it!,antoinepayet,2022-09-01 00:00:16
4458417,5923872,5504693,"RT @tboulanger: I'm buying new uggs tomorrow, Probly 2 pairs.... (Bbmthinkingface)<- dog, its march.. Give it up.",leclerchenri,2022-09-01 00:08:56


{{< /highlight >}}

La première ligne nous renseigne sur la façon dont sont stockées les données et ce qu'elles représentent. On observe que nous disposons d'un index, du texte (le tweet en lui-même), d'un pseudo (username), qui est la personne qui a publié le tweet, et de la date. Il y a environ 500 000 lignes, soit autant de tweet.

## Une aiguille dans une botte de foin

Comment s'y retrouver dans une telle masse de données ? Il y a clairement trop de tweet pour tous les lire, alors comment arriver à identifier **le** pseudo qui serait à l'origine d'une "campagne de désinformation" ? Il va faloir utiliser des outils adaptés et essayer d'être malin. En effet, face à autant de donnée, et dans un temps limité (la compétition durait une journée), il est essentiel d'aller droit au but et de ne pas ré-inventer la roue.

## Un panda et un serpent à la rescousse

Nous allons utiliser le langage `Python` et la librairie `pandas` [voir ici pour les curieux](https://pandas.pydata.org/). Cette librairie regroupe un ensemble d'outil très pratiques lorsqu'il s'agit de traiter de gros volumes de données. Même si on ne travaille pas sur des volumes de données incroyablement grands, l'optimisations et les outils déjà mis en place dans `pandas` vont nous en faire gagner.

{{< alert >}}
La suite contient du code. **Il n'y a pas besoin de s'y connaître en code pour comprendre l'article**. Le code est là pour satisfaire les profils plus techniques. Cet article n'est pas non plus un tutoriel pour apprendre à utiliser la librairie `pandas`.
{{< /alert >}}

## Une vue d'ensemble

Une première étape pour démêler tout ça est de connaître un peu mieux notre *dataset* (i.e : l'ensemble des données). Pour cela on peut faire quelques statistiques génériques, par exemples savoir combien de comptes sont présents dans notre jeu de données. Pour bien interpêter les résultats, il faut également se demander à quoi ils devraient ressembler. Qu'attendons-nous à obtenir ? 10, 10 000, 100 000 ? Intuitivement on devrait voir quelques comptes revenir mais le nombre de compte devrait être dans le même ordre de grandeur que celui des tweets.

{{< highlight python3 >}}
# on commence toujours par ouvrir le fichier csv, cette ligne sera induise par la suite
df = pd.read_csv('elections.csv', names=['n1','n2','index','text','username','date'])
number_of_accounts = len(df['username'].unique())
{{< /highlight >}}

Et là... surprise ! Il n'y a que 569 comptes... pour 500 000 tweets. C'est très étrange mais nous y reviendrons. Pour connaître un peu mieux encore notre dataset, on peut compter le nombre de tweets ainsi que le nombre **unique** de tweets, peut-être qu'il y a des tweets identiques ?

{{< highlight python3 >}}
number_of_unique_tweets = len(df['text'].unique())
number_of_tweets = len(df)
{{< /highlight >}}

Là encore, surprise : le nombre de tweet est d'environ 500 000 alors que le nombre de tweets **uniques** est de 243 000... **On a environ la moitié du contenu qui est dupliqué**, avec en prime un très faible nombre de compte. Cela semble indiquer un réseau de robot, mais rien de sûr, il va falloir creuser.

## Des robots partout ?

Un très petit nombre de compte, du contenu dupliqué, à priori on est en présence de robots qui publient beaucoup. Pour en être certain nous allons faire un peu de statistiques.

Premièrement, on peut s'intéresser au nombre moyen de tweets publiés par compte :

{{< highlight python3 >}}
average = df.mean()
{{< /highlight >}}

En moyenne un compte publie **903 tweets**. On pourrait penser que c'est énorme et pourtant cet indicateur est limité. En effet, une petite population de compte sur une longue période pourraient produire le même résultat sans pour autant héberger des robots. Plutôt que de s'intéresser à la durée de la capture des tweets, on va plutôt regarder si des comptes postent beaucoup tandis que d'autres non.

Pour cela on va utiliser **l'écart type** qui mesure l'écart à la moyenne, vous trouverez plus d'information ici : [Wikipédia : Écart type](https://fr.wikipedia.org/wiki/%C3%89cart_type). Pour résumer il va nous indiquer si des comptes publient beaucoup, tandis que d'autres moins. Un écart type important va indiquer une grande disparité d'activité entre les comptes et inversement.

{{< highlight python3 >}}
average = df.std()
{{< /highlight >}}

Le résultat confirme l'intuition ; l'écart type est de **4088**. Cela signifie que certains compte postent **énormément** par rapport à d'autre. Un écart type de 50 ou 100 aurait été la marque d'un réseau plus homogène, potentiellement composé exclusivement d'humains.

Si on affiche les 20 comptes les plus actifs, on se rend clairement compte qu'une grosse partie du traffic n'est pas naturelle :

{{< highlight python3 >}}
number_of_tweet_by_pseudo = df['username'].value_counts()
number_of_tweet_by_pseudo.head(20)
{{< /highlight >}}

{{< highlight txt >}}
r2d2x1977          30000
nonox1981          30000
mariax1927         30000
terminatorx1984    30000
goldorakx1975      30000
c3pox1977          30000
rodneyx2005        30000
wallex2008         30000
tinmanx1939        30000
robocopx1987       30000
cyber-rider        30000
valletyves           379
mathilde72           379
guerinmadeleine      378
jacquesvictor        376
gerard98             376
bertrandcarlier      376
hoareaususan         374
andree94             373
bonneauthibaut       372
{{< /highlight >}}

**Les 11 comptes les plus actifs ont exactement le même nombre de tweets et pire que cela, ils tweet 100 fois plus que les autres.**

## Des robots, mais pour quoi faire ?

Des robots, oui, mais pour quoi faire ? On nous a parlé de "campagne de désinformation", mais que disent-ils ces robots au juste ? Intéressons-nous à certains de leurs tweets :

{{< highlight python3 >}}
# affiche n tweets des pseudos ciblés
def display_tweets_from_bots(accounts, n, df):
    for username in accounts:
        print(username)
        i = 0
        for index, data in df.iterrows():
            if data['username'] == username and i < n:
                i+=1
                print(data['text'])
            elif i==n: break


number_of_tweet_by_pseudo = df['username'].value_counts()
# on a 11 comptes qui ont tous 30k tweets exactement
# on regarde ce qu'ils tweets
accounts = number_of_tweet_by_pseudo.head(11)
accounts = accounts.to_dict().keys()
display_tweets_from_bots(accounts, 5, df)
{{< /highlight >}}

{{< highlight txt >}}
r2d2x1977
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @GoldorakX1975 Dolore porro quiquia tempora.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
RT @WallEX2008 Numquam quiquia sed dolor.
nonox1981
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @WallEX2008 Numquam quiquia sed dolor.
RT @GoldorakX1975 Dolore porro quiquia tempora.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
mariax1927
Ut porro magnam labore est etincidunt numquam.
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @WallEX2008 Numquam quiquia sed dolor.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
RT @GoldorakX1975 Dolore porro quiquia tempora.
terminatorx1984
Dolore quaerat ut velit.
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @WallEX2008 Numquam quiquia sed dolor.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
RT @GoldorakX1975 Dolore porro quiquia tempora.
goldorakx1975
Quisquam neque modi ipsum magnam est porro amet.
Dolore porro quiquia tempora.
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @WallEX2008 Numquam quiquia sed dolor.
c3pox1977
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @GoldorakX1975 Dolore porro quiquia tempora.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
RT @WallEX2008 Numquam quiquia sed dolor.
rodneyx2005
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
RT @GoldorakX1975 Dolore porro quiquia tempora.
RT @WallEX2008 Numquam quiquia sed dolor.
wallex2008
Numquam quiquia sed dolor.
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
RT @GoldorakX1975 Dolore porro quiquia tempora.
tinmanx1939
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
RT @WallEX2008 Numquam quiquia sed dolor.
RT @GoldorakX1975 Dolore porro quiquia tempora.
robocopx1987
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
Sed sed sit sed ipsum.
RT @WallEX2008 Numquam quiquia sed dolor.
cyber-rider
RT @TerminatorX1984 Dolore quaerat ut velit.
RT @MariaX1927 Ut porro magnam labore est etincidunt numquam.
RT @WallEX2008 Numquam quiquia sed dolor.
RT @GoldorakX1975 Dolore porro quiquia tempora.
RT @GoldorakX1975 Quisquam neque modi ipsum magnam est porro amet.
{{< /highlight >}}

On observe qu'ils produisent tous du contenu... en latin. Les comptes plus naturels, eux, restent en anglais. Cependant, il semblerait quand dans les tweets que nous avons mis au jour, il y ait beaucoup de retweet (RT). De plus, ces RT concernent systématiquement d'autres membres du réseau. **En clair, ils se retweet entre-eux.** Passons outre le latin et intéressons-nous un peu à ce comportement.

## Réseautage et amplification

Manifestement, la solution se trouve dans ces 11 comptes. La supposition peut sembler infondée et plus tenir de l'intuition que d'une méthode rigoureuse, j'y reviendrai à la fin de l'article.

Si l'on considère ces 11 comptes comme faisant partie d'un réseau, on peut étudier les relations en son sein. Quelles sont les relations entre ces comptes ? Que font-ils exactement ?

La seule relation que deux comptes peuvent avoir est le RT. De plus, on peut savoir si un compte a RT un tweet d'un autre compte, lui aussi appartenant au réseau. Ce faisant, on peut comprendre un peu mieux la structure du réseau et ce que les bots y font.

{{< highlight python3 >}}
def network(accounts, df):
    for username in accounts:
        inside_network, outside_network, not_rt = 0, 0, 0
        for index, data in df.iterrows():
            # tweet d'un bot
            if data['username'] == username:
                # rt d'un bot
                if '@' in data['text']:
                    has_bot = False
                    for a in accounts:
                        if a.lower() in data['text'].lower():
                            has_bot = True
                    if has_bot:
                        inside_network += 1
                    else:
                        outside_network += 1
                else:
                    not_rt += 1
        print(username, 'inside network', inside_network, 'outside network', outside_network, 'not retweet', not_rt)
{{< /highlight >}}

Cette fonction va afficher, pour chaque robot, le type de relation entretenue avec les autres membres.

{{< highlight txt >}}
r2d2x1977 inside network 27000 outside network 0 not retweet 3000
nonox1981 inside network 27000 outside network 0 not retweet 3000
mariax1927 inside network 27000 outside network 0 not retweet 3000
terminatorx1984 inside network 27000 outside network 0 not retweet 3000
goldorakx1975 inside network 27000 outside network 0 not retweet 3000
c3pox1977 inside network 27000 outside network 0 not retweet 3000
rodneyx2005 inside network 27000 outside network 0 not retweet 3000
wallex2008 inside network 27000 outside network 0 not retweet 3000
tinmanx1939 inside network 27000 outside network 0 not retweet 3000
robocopx1987 inside network 27000 outside network 0 not retweet 3000
cyber-rider inside network 30000 outside network 0 not retweet 0
{{< /highlight >}}

On observe que tous les membres du réseau restent entre-eux. **Ils ne communiquent pas à l'extérieur de leur bulle**. Il y a très peu de contenu produit, la majorité étant republié (que 3000 tweets qui ne sont pas des re-publications), ce qui corrobore ce qui a été dit plus haut. Enfin, il n'y a qu'un seul compte qui est différent des autres : **cyber-rider**. Celui là ne re-publie rien mais tout ce qu'il produit est relayé dans le réseau. **En d'autres termes, il produit le contenu qui est ensuite amplifié artificiellement au sein du réseau. C'est le compte qui est à l'origine de la "campagne de désinformation".**

{{< alert >}}
cyber-rider est la réponse au challenge
{{< /alert >}}

Pour résumer, dans la masse de contenu, se dessine un réseau de 11 comptes dont l'un d'eux se contente de produire du contenu tandis que les autres le partage entre-eux.

## Lutte d'influence informatique

Revenons un peu sur la thématique globale du challenge. Ce sujet de lutte d'influence informatique, abrégé L2I, monte depuis plusieurs années. [Le Ministère des Armées la définit comme suit](https://www.defense.gouv.fr/ema/actualites/armees-se-dotent-dune-doctrine-militaire-lutte-informatique-dinfluence-l2i) :

> La lutte informatique d’influence (L2I) désigne les opérations militaires conduites dans la couche informationnelle du cyberespace pour y détecter, caractériser et contrer les attaques, renseigner ou faire de la déception, de façon autonome ou en combinaison avec d’autres opérations.

Ce type d'attaque, nombre de pays y sont confrontés, dont la France et plus particulièrement dans le cadre d'opérations militaires au Sahel. Ces opérations de désinformation visent à salir l'image de la France en Afrique au profit d'autre puissance, comme la Russie. Tous les pays et toutes les organisations peuvent être touchés par cette nouvelle forme d'agression. Pour plus d'information, je vous invite à consulter ces ressources : 
- [Courrier International. 2022. "Guerre de l’ombre. En Afrique de l’Ouest, l’offensive des réseaux russes de désinformation"](https://www.courrierinternational.com/article/guerre-de-l-ombre-en-afrique-de-l-ouest-l-offensive-des-reseaux-russes-de-desinformation).
- [Arte. "Guerre de l'info: au coeur de la machine Russe"](https://www.youtube.com/watch?v=HlMCoJOy8XU)
- [Editions Puf "Les guerres de l'information à l'ère numérique". 2021. Céline Marangé & Maud Quessard.](https://www.puf.com/content/Les_guerres_de_linformation_%C3%A0_l%C3%A8re_num%C3%A9rique)

L'un de mes projets d'étude porte sur cette thématique et j'espère pouvoir écrire dessus un jour. Ce sujet me tiens à coeur, il est vital pour nos démocraties et riche de connaissance et de découverte.

## Critique

Vous l'aurez remarqué, la notion de désinformation est toute relative. Il n'empêche que l'on peut se questionner sur la portée d'une désinformation qui, de toute évidence, n'est rien d'autre que du *Lorem Ipsum*. Ce texte en latin ne vise ni le personnage mentionné dans l'énnoncé ni le pays fictif. Il aurait peut-être été plus intéressant d'être plus proche de la réalité, avec un réseau de robot moins bruyant et plus efficient. Cependant, ce challenge a été l'occasion de s'initier à la data science, la L2I et l'utilisation d'outils tout en apportant un style de challenge dont on a peu l'habitude.