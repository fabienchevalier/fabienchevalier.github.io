---
title: "Projet d'étude Master 2024 - Golang, API et IAC"
date: 2024-06-10
description: "Réalisation d'une API en Go proposant une interface REST permettant la gestion d'une chaîne de certification."
showComments: True
showTableOfContents:
  article: True

draft: True
---
{{< lead >}}
Ce projet à été réalisé dans le cadre de ma première année de Master, en collaboration avec [Matteoz](https://gitlab.com/Toxma). Je me suis tout particulièrement intéressé à la partie `DevOps` du projet, à savoir la mise en place d'une pipeline de livraison continue ainsi que la partie infrastructure as code permettant d'héberger l'applicatif.
{{< /lead >}}

## Introduction

Le but de cet article est de résumer la façon dont j'ai appréhendé l'intégration d'un applicatif au sein d'une infrastructure dite `cloud-native`. Ce projet couvrais le développement d'une API REST, jusqu'à son déploiement de façon continue sur un environnement AWS imposé, à savoir [AWS ECS](https://docs.aws.amazon.com/fr_fr/AmazonECS/latest/developerguide/Welcome.html). S'agissant d'un travail en groupe de deux, la répartition de la charge de travail à été établie comme ceci :

- [Matteoz](https://gitlab.com/Toxma) c'est chargé de la partie développement applicative
- Pour ma part, j'ai mis en place une usine logicielle permettant à mon collègue de déployer l'application sur une infrastructure managée par le code.

Le sujet est disponible [ici](sujet.pdf).

L'ensemble du code source et la documentation projet est disponible sur le [groupe GitLab du projet](https://gitlab.com/gocert).

{{< alert "circle-info" >}}
Cet article n'as pas pour but de faire doublon avec la doc technique du projet. Il s'agit ici de résumer dans les grandes lignes la façon dont j'ai appréhendé la réalisation de ce projet
{{< /alert >}}

### Problématique

Comme c'est le cas dans une situation pro, le sujet fourni est une forme d'expression des besoins ayant pour finalité de répondre à une problématique. Le contexte est donc le suivant : 

- Une société appelée `Solution Libre` propose des services internes et externes pour lesquels les problématiques de chiffrement sont essentiels.
- Pour répondre à ses besoins en la matière, une API de gestion de certificats doit être développée.

Les fonctionnalités attendues sont indiquées dans le sujet à savoir :

- Un CRUD permettant la gestion des droits
- La génération de certificats
- Leur gestion (CRUD)
- Leur vérification (y compris les certificats externes)

Côté tooling, certaines choses sont imposées :

- Golang pour le language de programmation
- GitLab pour l'hébergement de code source + CI/CD
- AWS ECS pour la partie hébergement
- La solution fournie doit être livrée sous forme de microservice
- Niveau backend

### Zoom sur ma partie, l'intégration

J'ai 