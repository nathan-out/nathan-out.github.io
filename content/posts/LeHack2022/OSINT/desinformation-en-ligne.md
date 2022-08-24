---
title: "Cactées et succulentes, anatomie de la désinformation (Qactus)"
date: 2022-06-30T13:56:22+02:00
draft: false
---
*[Open Facto](https://openfacto.fr/)* fédérer l'OSINT français

**Qactus** : site de désinformation français proche de la mouvance Qanon (USA), les thématiques abordées sont les complots en tout genre, l'antivaccination, l'antisémitisme... Il s'est fait connaître durant la crise du COVID. Le compte Twitter du site a été banni mais ce réseau social reste un lieux privilégié pour la diffusion des articles.

Technologiquement, c'est un blog WordPress créé le 6 mai 2020 et qui comporte un traqueur publicitaire (détail important pour la fin). Le but de la conférence est de comprendre l'environnement de ce genre de site, son fonctionnement ainsi que d'essayer de retrouver son/ses créateur(s)/administrateur(s).

Les pivots utilisés lors de l'enquête :
- [iqwhois](https://iqwhois.com/)
- [archive.org](https://archive.org/)
- [publicwww](https://publicwww.com/) (moteur de recherche pour du code source de site)

## Comprendre l'écosystème

Les outils utilisés sont :
- [Hyphe](https://hyphe.medialab.sciences-po.fr/) : collecte et cartographie de l'information. Une fonctionnalité intéressante est de pouvoir calculer la densité d'un graphe ; permet de savoir quel site cite t
- [Gephi](https://gephi.org/) : analyse et cartographie de l'information (très complémentaire avec Hyphe)
- Retina (lien de l'outil ?) : visualisation de la donnée
- [Smat.app](https://www.smat-app.com/) : visualisation dans le temps selon des critères, par exemple on peut voir l'impact de la désinformation par Qactus pendant et après le covid.
- [Snscrape](https://github.com/JustAnotherArchivist/snscrape) : récupération de contenus des réseaux sociaux (ici Twitter)
- [Open Refine](https://openrefine.org/) : nettoyage et mise en forme des données.
- L'analyse du corpus de donnée a été faite avec python ou le langage R.

Une fois la donnée collectée et raffinée, on observe ceci : 
- 120k tweets comportant un lien vers qactus ont été publiés depuis le 26 juin 2020.
- 9k comptes différents ont intéragis avec au moins un post
- certains utilisateurs sont très actifs (ex : 3.9k tweets pour le plus actuf)
- certains utilisateurs sont influents (ex : Christine Kelly & Di Borgo)
- la communauté est relativement petite mais très active

Après analyse des liens du site, on observe bien que Qactus est proche de la mouvance Qanon ; beaucoup de liens renvoient vers ce site ou d'autres du même genre. On observe une bulle de filtre.

On peut également dresser le palmarès des plus gros posteurs, retweeters, likers...

## Qui est derrière le site ?

Un site sous WordPress permet de récupérer la liste des auteurs et d'autres informations utiles. On récupère un pseudo qui nous ammène vers un compte Google et une chaîne Youtube. Cela semble être la chaîne officielle de Qactus. Si on recherche dans les leaks, les pages jaunes ou copain d'avant on trouve quelques informations en plus et des noms qui pourraient être la véritable identité de créateur du site. Grâce à [pappers.fr](https://www.pappers.fr/) on peut retrouver la SARL d'un candidat, sa date de naissance, sa signature... [Wayback machine](https://archive.org/web/) nous permet de retrouver un vieux site de 2004, ce qui semble indiquer que la personne possède des compétences en informatique malgré le fait qu'il ait environ 50-60 ans. On tombe sur une adresse postale qui correspond à l'une de celles trouvées précédemment, on a une quasi-certitude sur l'administrateur du site internet. Certaines informations permettent également de remonter jusqu'à sa compagne, qui semble être dans le même système de pensée, cela renforce les résultats.

Les auteurs de la conférence ont essayé de contacter cette personne pour avoir une confirmation, sans succès. Néanmoins ils sont quasi-certains qu'il s'agit de cette personne.

Si on revient au traqueur publicitaire découvert en début d'enquête, on peut estimer que les revenus publicitaires sont d'environ 30 à 40k€/an. Au delà de l'aspect politique, la désinformation est également une activité économique qui peut s'avérer très rémunératrice.

## Conclusion

- Qactus évolue dans une bulle clairement identifiable, son créateur et principal administrateur l'est aussi
- Propagation de l'information sur des réseaux "mainstream" comme "sous-terrain" ex : Gettr (réseau social américain conservateur)
- communauté forte et active
- participe à saturer l'espace publique (selon les mots du conférencier)
- un site comme celui-ci peut susciter un intérêt financier
- il est relativement simple d'analyser finement une aussi grande sphère d'information
