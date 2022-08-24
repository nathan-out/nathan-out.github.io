---
title: "Logic flaws, what are we missing in web application"
date: 2022-06-30T13:56:22+02:00
draft: false
---
*Mirza Burhan Baig*

**<u>Logic flaw</u>** : traduit en "faille logique" dans ce retex, est une faille qui vient d'un manquement de l'application à vérifier son comportement lorsque l'utilisateur effectue des actions innatendues.

Le top 10 des vulnérabilités web OWASP répertorie ce type de faille dans son classement. Pourtant, cela reste largement sous testé.

Ce type de faille vient d'une confiance excessive que l'application a envers l'utilisateur, ou la partie cliente d'une application. La faille résulte d'un échec du traitement d'une entrée non conventionnelle. Il faut partir du principe que l'utilisateur ne va pas toujours suivre le chemin prévu, surtout lorsqu'il s'agit d'une personne malveillante. De manière générale, la faille résulte d'une néglicence, d'un manquement dans le process.

Ce type de faille peut être aussi dévastatrice qu'une XSS ou une injection SQL. Il ne faut pas les sous-estimer ou les ramener à de simples bugs, mais plutôt les considérer comme des failles à part entière. Les failles logiques sont plutôt présentes dans des applications bancaires, du fait de leur complexité ainsi que des multiples étapes dans les processus d'authentification.

<br>

**<u>Plusieurs pistes pour gérer ces failles :</u>**
- ne pas faire confiance à l'utilisateur ou à la partie cliente d'une application.
- implémenter des jeux de tests complets dans l'application.
- audit en boîte blanche et analyse de la gestion des erreurs, des réponses de l'application ainsi que les points critiques (authentification par exemple).
