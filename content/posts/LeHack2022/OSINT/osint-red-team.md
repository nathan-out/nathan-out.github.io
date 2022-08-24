---
title: "L'OSINT dans la Red Team"
date: 2022-06-30T13:56:32+02:00
draft: false
---

*Epios*

<u>Objectif :</u> : compromettre une organisation. On utilisera par exemple du crochetage, de l'USB dropping... La **cyber kill chain** est la suite d'étape permettant de compromettre une organisation.

## Reconnaissance via l'OSINT

On différencie bien l'OSINT de l'ingénierie sociale. L'objectif est de passer innaperçu. Pour cela, l'OSINT peut nous aider à connaître le vocabulaire propre à l'entreprise, les employés immportants/intéressants, la connaissance de prestataires de l'entreprise mais aussi les lieux stratégiques. Le présentateur montre deux exemples : 
- dorks pour connaître certains prestataires de la CIA.
- geoint pour deviner l'emplacement de datacenter interne à l'entreprise (via la climatisation notamment) et les points de livraisons, souvent laissés ouverts et sans surveillance particulière.

Les réseaux sociaux ont une importance, on a tendance à les sous-estimer et à ne pas tous les regarder. En effet, les stagiaires et/ou jeunes employés sont sur **Snapchat, Tiktok ou Instagram**, ils peuvent par exemple filmer une partie de leur quotidien dans l'entreprise et nous informer sur la structure des locaux, le matériel utilisé, la présence ou non de badgeuse... De la même manière, il faut faire attention aux **employés** ainsi qu'aux **visiteurs**.

Le présentateur illustre ses propos par l'affaire Strava (*Jean-Marc Manach, journaliste chez Next INpact*) : des militaires/espions qui utilisaient cette application de running et qui permettait de découvrir des bases militaires secrètes. Strava est un petit réseau social, on peut par exemple y faire des groupes de coureur. Cela peut être très utile pour trouver des noms d'employés et voir leurs habitudes de courses (à priori cette fonctionnalité est en train d'être patchée).

A partir de toutes ces informations, il est possible d'approcher des personnes de l'entreprise et de gagner leur confiance. Il est alors possible de les manipuler, on connaît leurs passions, son niveau d'accès dans l'entreprise voire même son tempérament.

Quelques outils et références :
- [The Stuxnet Story](https://www.youtube.com/watch?v=Joc0iTX9dyQ&ab_channel=LangnerGroup) -> de l'OSINT a été utilisé à partir d'images de propagande du régime iranien pour connaître le type d'équipement, les versions des logiciels...
- [Crystalknows](https://www.crystalknows.com/) -> IA qui, à partir d'un compte LinkedIn, permet de dresser un profil psychologique d'une personne.
- [Hunter.io](https://hunter.io/) -> trouver des adresses email professionnelles.
- [Pimeyes](https://pimeyes.com/) -> (payant) reconnaissance de personnes à partir d'une photo de leur visage.

L'étape suivante consiste à passer à l'attaque.
