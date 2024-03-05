---
title: "Machine Learning x Cybersécurité : analyse d'URLs frauduleuses"
date: 2024-03-04T14:06:22+02:00
draft: false
---

> Article décliné du write-up issu du challenge *Diabolical Grumpy Analyst* du [GCC CTF 2024](https://gcc-ctf.com/).

## Résumé

Ce challenge montre que les professionnels de la cybersécurité devraient se saisir des outils de *Machine Learning* pour construire des solutions plus efficientes.

En effet, il est possible de construire des modèles rudimentaires mais efficaces à partir de données. Le langage `Python` allié avec des librairies comme `Pandas` et `Scikit-learn` permettent d'accéder à ces capacités sans formation particulière. Ces modèles peuvent répondre à des besoins ponctuels, tels que la détection de fraude ou la pré-classification d'information.

Cependant, lorsque les prédictions des modèles peuvent avoir un impact fort, il convient d'apporter le regard critique d'un expert. Des phénomènes tels que le surapprentissage ou la présence de biais dans les données peuvent fausser les résultats. Détecter ces pièges nécessite des connaissances poussées et une véritable expertise.

## Le challenge

L'objectif du challenge est d'interroger un serveur via `netcat`, de récupérer une liste d'URL, de les classifier (légitimes ou malveillantes) et de renvoyer le résultat au serveur. Si on a plus de 85% de précision, alors on obtient le flag.

Pour cela, on dispose de deux fichiers qui comportent respectivement des URLs saines et malveillantes. Ce sont des fichiers textes où chaque ligne comporte une URL.

La manière la plus simple, à mon sens, pour résoudre le challenge est d'utiliser le *Machine Learning*. L'idée est que, d'après les deux fichiers textes, on puisse entraîner un modèle pour classifier les URLs. La question est : quel type de modèle choisir dans cette situation ?

{{< alert >}}
Le choix d'un modèle se fait toujours par rapport à une situation précise. Il n'existe pas (encore ?) un modèle général qui pourrait répondre à toutes les problématiques.
{{< /alert >}}

### Choisir un modèle

Un modèle peut faire toute sorte de choses : générer de la donnée, la classifier, détecter des patterns... Et il peut être **suppervisé** ou **non suppervisé**, cela indique si nos données sont labellisées ou non. Dans notre cas, nous devons **classifier** des URLs dans deux catégories : saines et malveillantes et nos données d'entraînement sont labellisées.

Il nous faut donc un algorithme de **classification binaire supervisé**. Ça tombe bien, c'est justement le plus simple lorsqu'on débute ! Il existe plusieurs algorithmes répondant à ces prérequis et dont les usages peuvent être un peu différents mais on ne rentrera pas dans le détail, plus d'information ici : [Top 10 Binary Classification Algorithms [a Beginner’s Guide]](https://towardsdatascience.com/top-10-binary-classification-algorithms-a-beginners-guide-feeacbd7a3e2).

Un des modèles de classification binaire les plus simple à utiliser est un **arbre de décision** ou *Decision Tree Clasifier*. L'idée d'un arbre de décision est assez intuitive. Même si l'algorithme est intégré à `Scikit-learn` (on ne le codera pas nous-mêmes), il est important de comprendre les grandes lignes.

### Decision Tree Classifier

On va envoyer des données, **sous forme de nombre** (retenez bien ça), à l'arbre et il va classer cette donnée dans une des deux catégories. Au début, il va le faire de manière plus ou moins aléatoire. Ensuite, on va lui indiquer si sa classification était correcte ou non (car on a des données labellisées). L'arbre va ensuite se réorganiser en fonction de son résultat (on ne rentrera pas dans le détail de la façon dont il le fait). On notera bien que l'arbre se réorganisera, qu'il ait correctement classifié ou non c'est seulement la manière de se réorganiser qui changera. Au fur et à mesure que les données d'entraînement passent dans l'arbre, ce dernier se réorganise et la façon dont il le fait renforce la précision de ses résultats.

> Pour les matheux, on peut interpréter ce que l'arbre fait comme ceci : il créé empiriquement une fonction mathématique qui découpe un ensemble de points en deux ensembles.

### Comment on fait, concrètement ?

Maintenant que nous savons quel modèle utiliser et la façon dont il fonctionne, passons à la pratique. Notre script va suivre 4 étapes :

1. pré-processing des données

2. entraînement du modèle

3. évaluation (calcul de sa précision)

4. utilisation du modèle en cas réel

#### Pré-processing

Ou un grand mot qui fait peur pour dire simplement qu'on va transformer nos URLs en nombre. On se rappelle que le modèle ne travaille **que sur des nombres**. En fait, on va extraire des statistiques numériques, ou *features* de chaque URLs.

On pourrait calculer pleins de *features* différentes qui auraient un impact plus ou moins important sur la précision des résultats. On pourra s'amuser avec le modèle en rajoutant/supprimant des *features* une fois qu'il sera oppérationnel.

Voici les *features* que j'ai choisi de calculer : 

- longueur de l'URL

- nombre de sous-domaines (monsite.tech.fr en a 2 alors que monsite.fr n'en a qu'un)

- nombre de caractère spéciaux (/, ., -, _)

- longueur du domaine

- nombre de tirets

- nombre de chiffres

On pourrait en ajouter pleins d'autres...

```python
def extract_features(url):
    import urllib.parse
    parsed_url = urllib.parse.urlparse(url)
    length = len(url)
    num_special_chars = sum(1 for char in url if char in ['/', '.', '-', '_'])
    num_subdomains = len(parsed_url.netloc.split('.'))
    len_domain = len(url.split('.')[0])
    len_subdomain = len(url.split('.')[1])
    num_digits = count_digits(url)
    num_hyphen = count_car(url, '-')
    return [length, num_special_chars, num_subdomains, len_subdomain, len_domain, num_digits, num_hyphen]
```

#### Entraînement et évaluation

Juste avant d'entraîner le modèle, il faut comprendre ce qu'il attend. Il souhaite une liste de données numériques et un label associé (malveillant ou sain). Par convention, on appelle `X` les données numériques et `y` les labels associés. Il nous faut donc deux listes. 

{{< alert >}}
Les données d'entraînement a envoyer au modèle, `X`, doivent être une liste 2D !
{{< /alert >}}

Pour mieux comprendre, voici un exemple des données à envoyer au modèle (`X`) pour l'entraînement :

```python
[
    [24, 3, 1, 2, 22, 4, 0], # URL numéro 1
    [12, 7, 1, 1, 10, 7, 1], # URL numéro 2
    ...
]
```

Et un exemple des labels associés `y` :

```python
[
    0, # signifie "légitime"
    0,
    1, # signifie "malveillante"
    ...
]
```

Le script complet :

```python
X, y = [], []

for url in legit_urls:
    X.append(extract_features(url))
    y.append(0)

for url in dga_urls: # URLs malveillantes
    X.append(extract_features(url))
    y.append(1)
```

Pour évaluer la précision du modèle, on va découper les données d'entraînement en deux :

- une partie qui servira à l'entraînement du modèle

- une partie pour l'évaluer

{{< alert >}}
On n'évalue **jamais** un modèle sur des données qu'il a vu à l'entraînement.
{{< /alert >}}

`Scikit-learn` nous fournit une fonction bien pratique pour faire ça (`test_size` signifie que 20% des données seront utilisées pour le test et 80% pour l'entraînement, c'est une proportion courante) :

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
```

Enfin il ne reste qu'à créer le modèle, l'entraîner et évaluer sa précision :

```python
from sklearn.tree import DecisionTreeClassifier
from sklearn.metrics import accuracy_score

model = DecisionTreeClassifier()
model.fit(X_train, y_train) # entraînement avec les données d'entraînement
predictions = model.predict(X_test) # prédiction sur les données de test
accuracy = accuracy_score(y_test, predictions) # calcul de la précision par rapport au prédictions
print("Exactitude (Accuracy) :", accuracy)
```

A cette étape là, on a ~80% de précision, ce qui est insuffisant puisque le challenge demande 85%. On reviendra plus tard sur la façon dont on peut l'améliorer.

#### Utilisation

Si vous avez compris comment l'entraîner, alors cette partie devrait être simple puisqu'on va suivre les mêmes étapes de pré-processing.

```python
urls = [ # les URLs à envoyer au modèle
    'buijmmnvpolshlijbuzeols.org', # une URL qu'on pourrait raisonnablement qualifier de malveillante
    'laboratorieswi.com' # une URL plutôt légitime
]

data_to_predict = [] # on n'oubile pas que le modèle attend un tableau 2D, comme lors de l'entraînement
for url in urls:
    data_to_predict.append(extract_features(url))

print(model.predict(data_to_predict)) # la fonction predict retourne une liste de classe, ici 1 ou 0 en fonction de la façon dont il a classifié les URLs.

# Le résultat attendu est : [0, 1]
```

### Amélioration

Pour passer la barre des 85%, on peut rajouter des *features*. Dans la contrainte de temps du CTF, j'ai rajouté des statistiques à la vollée pour résoudre rapidement le challenge. J'ai essentiellement compté le nombre de "a", "b", "x", "y" et "z". Cela a suffi pour gagner les quelques pourcents restant.

Je détaille dans la partie suivante pourquoi ce que j'ai fais **n'est pas conseillé**.

## Approche critique

Tout d'abord je précise que je ne suis **pas** un *Data Scientist* ni un professionnel du *Machine Learning*. Je suis spécialisé en cybersécurité avec néanmoins un attrait et une petite formation en ML.

Vous aurez noté que j'améliore artificiellement les résultats en ajoutant des *features*. Ces dernières sont capitales pour l'entraînement mais également la fiabilité du modèle. De mauvaises *features* auront tendance à produire de mauvais résultats, ou pire, des résultats biaisés. Je ne suis pas certains que compter le nombre d'occurence de lettre soit une statistique intéressante. Cependant, des méthodes existent pour tenter de dégager les bonnes des mauvaises *features*.

Les biais dépendent directement de la **qualité** des données d'entraînement. Les données doivent être nombreuses, variées et représentatives autant que possible des cas réels.

Il faut également faire attention au **surapprentissage**. Cela intervient lorsque le modèle apprend "par coeur" les données d'entraînement. Il va alors parfaitement les intégrer mais va perdre en capacité de généralisation. Si bien, que le modèle va sous-performer sur les données du cas réel. Il existe également des bonnes pratiques pour limiter l'apparition de ce phénomène, ainsi que des métriques pour le mesurer.

Enfin, et c'est peut-être là le plus important, les modèles de *Machine Learning* restent des **outils**. Cela peut paraître trivial, mais ça va mieux en le disant. Il faut adopter une approche critique de leurs résultats, comprendre leurs fonctionnement pour mieux appréhender leurs limites et cas d'usages.

Le *Machine Learning* n'a rien de magique ni de mystérieux, cela va de soit en le disant. Néanmoins, même si le fonctionnement est parfois obscur et l'explicabilité des résultats complexe, ces modèles restent des **modèles statistiques**, avec toutes les limitations et la puissance qu'on leur connaît.
