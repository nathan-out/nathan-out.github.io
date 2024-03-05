---
title: "GCC-CTF 2024 · Forensique  · BipBipBiiip"
date: 2024-03-04T13:06:22+02:00
draft: false
---

> Write-Up issu du challenge *BipBipBiiip* du [GCC CTF 2024](https://gcc-ctf.com/).

L'objectif du challenge est de trouver des anomalies dans une liste de numéros de téléphone
dont les formats sont... divers et exotiques :

```
+33 7 90 46 14 42
432.598.1258x6
543-584-559-0024x854
05 58 93 42 59
090-8945-0526
(576)358-1547
275-543-3517
0483461831
03-1255-1140
254.315.2697
...
```

Tous les formats ci-dessus sont des numéros de téléphones **valides**. Je suis parti d'une hypothèse : les données à trouver sont *significativement* différentes des formats de numéros de téléphone. Par *"significativement différentes"* j'entends qu'on ne vérifie pas si les numéros ont un chiffre en trop ou en moins, si l'indicatif (+33) est correct etc...

## Filtrage avec les regex

Je vais appliquer des filtres (*regex* ou *expression régulière*) successifs pour supprimer les formats de numéros de téléphones et à chaque filtre appliqué, j'affiche les quelques premiers numéros restants ainsi que leur nombre. Cela me permet de vérifier que mes filtres sont valides et de détecter d'éventuels bugs dans mes regex. J'utilise la librairie `pandas` par commodité :

```python
import pandas as pd

def apply_patterns(pattern_list, df):
    filtered_df = df['PHONE_NUMBER']
    print('number of number', len(filtered_df))
    for pattern in pattern_list:
        filtre = filtered_df.str.contains(pattern)
        filtered_df = filtered_df[~filtre]
        #print(df[~df['PHONE_NUMBER'].str.match(pattern)]['PHONE_NUMBER'].head(1))
        print('number of number', len(filtered_df), '\n')
    return filtered_df

df = pd.read_csv('phonebook.csv')
pattern_list = [
    r'\d{3}\.\d{3}\.\d{4}x\d+', # 432.598.1258x6
    r'\+\d{2} \d \d{2} \d{2} \d{2} \d{2}', # +33 7 90 46 14 42
    r'\d{3}-\d{3}-\d{3}-\d{4}x\d+', # 543-584-559-0024x854
    r'\+\d{2} \(\d\)\d \d{2} \d{2} \d{2} \d{2}',
    r'\d{2} \d{2} \d{2} \d{2} \d{2}', # 05 58 93 42 59
    r'\d{3}-\d{4}-\d{4}', # 090-8945-0526
    r'\(\d{3}\)\d{3}-\d{4}', # (576)358-1547
    r'\d{3}-\d{3}-\d{4}', # 275-543-3517
    r'\d{10}', # 0483461831
    r'\d{2}-\d{4}-\d{4}', # 03-1255-1140
    r'\d{3}\.\d{3}\.\d{4}', # 254.315.2697
]
filtered_df = apply_patterns(pattern_list, df)
print(filtered_df)
```

{{< alert >}}
Il y a plus efficace comme méthode mais l'application successive de filtres me permet d'éliminer les numéros valides et de progressivement détecter les anomalies. La librairie `pandas` n'est pas obligatoire mais permet de gagner du temps à l'exécution si on souhaitait industrialiser le processus.

Les numéros avec un "x" sont valides : ils indiquent que le numéro a une extension et cela est utile pour les standardistes.
{{< /alert >}}

Progressivement, le nombre de numéros restant se réduit et il ne reste que 4 lignes :

```
number of number 10000
number of number 9604 
number of number 8764 
number of number 8372 
number of number 7537 
number of number 6687 
number of number 4137 
number of number 3512 
number of number 2214 
number of number 1075 
number of number 279 
number of number 4 

3358      4743437b5233
3557    6733785f347233
3838        5f57316c64
5852        212121217d
```

Ces 4 dernières lignes sont en fait de l'hexadécimal, en décodant on obtient le flag : `from hex GCC{R3g3x_4r3_W1ld!!!!}`.
