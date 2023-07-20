---
title: "Hackaton Sup De Vinci 2023"
date: 2023-07-03
description: "Réaliser l'infrastructure permettant d'héberger une application web (front end/backend)"
showComments: False
showTableOfContents:
  article: True

draft: True
---
{{< lead >}}
Réalisé dans le cadre de mon année de Bachelor, le Hackaton à pour but de proposer une solution informatique en une 
semaine. Un jury juge ensuite la solution la plus convaincante et désigne les vainqueurs.

L'ensemble du projet que j'ai réalisé est disponible en open-source  
[sur GitLab](https://gitlab.com/sdv-open-course-factory)
{{< /lead >}}


## Introduction

Lors de ce hackathon, mon équipe à choisi pour sujet le développement d’un frontend via l’intégration de différentes technologies et la mise en 
place d’une plateforme SaaS avec toutes les contraintes que ça comporte (passage à l’échelle d’un PoC vers 
une plateforme réelle). Le projet a pour objectif de résoudre plusieurs problèmes liés aux supports de formation, à la 
fois côté conception mais aussi pour leur conservation et adaptation côté école.

### Contexte et travail d'équipe

L'ensemble des étudiants de l'école sont réunis en équipe de 10, tout niveaux et spécialités confondues. Toute la 
difficulté étant de coordonner l'ensemble de l'équipe afin de livrer un produit fonctionnel. Ainsi, il était demandé
de : 

- Réaliser un front-end applicatif en utilisant un backend fourni par l'école
- Réaliser l'infrastructure hébergeant le back et le front
- Fournir une fonctionnalité de livraison et d'intégration continue (CI/CD) 

### Limites et difficultés

Malheureusement, l'équipe dont j'ai fais partie n'as pas sû délivrer un front-end dans les temps impartis. Cela dit,
de mon côté j'ai pu architecturer l'ensemble de l'infrastructure ainsi que la pipeline CI/CD et obtenir un POC 
fonctionnel. Il ne me manquais plus qu'un front-end afin d'obtenir le résultat demandé, dommage :disappointed_relieved:!

Cependant, j'ai pu tester le fonctionnement du back-end sur l'architecture ainsi déployée. **C'est d'ailleurs l'objet de
cet article** : présenter ma solution sur la partie DevOps :sunglasses:. 

## Schéma d'architecture

J'ai réalisé un petit brouillon de l'architecture que j'ai souhaité mettre en place sur le cloud Azure : 

![Scheme](imgs/aks.drawio.png)

On à donc:

- Un cluster Kubernetes managé sur le cloud Azure (AKS)
- 3 namepsaces : un pour le front, un pour le back et le dernier pour le monitoring et la gestion des logs

{{< alert >}}
Lors de la réalisation, j'ai regroupé tout mes `pods` K8S au sein du même `namespace` par manque de temps. Ce projet
n'est de plus pas du tout destiné à être mis en production. Il manque d'ailleurs aussi toute la partie 
monitoring avec Elasticsearch/Prometheus.
{{< /alert >}}

En l'état, l'architecture disponible sur le lien GitLab donné en préambule permet uniquement de requêter l'API du
backend. De plus, comme indiqué sur le schéma la base de donnée utilisée est hébergée dans un `pod`. J'aurais préféré
mettre en place une base de donnée managée mais n'étant pas encore très à l'aise avec Kubernetes sur Azure, j'ai préféré
aller au plus simple.

### IAC : Terraform

L'enjeu étant de provisioner tout cela as code, j'ai déployé cette architecture avec Terraform. Le repo contenant
l'infrastructure est construit comme ceci:

```
├── README.md
└── terraform
    ├── environment
    │   └── dev
    │       └── variables.tfvars
    ├── main.tf
    ├── modules
    │   └── kubernetes
    │       ├── main.tf
    │       ├── outputs.tf
    │       └── variables.tf
    ├── outputs.tf
    └── variables.tf
```

Cette architecture est dès le départ conçue pour être en mesure de déployer la même infrastructure en fonction de
l'environement de production choisi. Pour la maquette, j'ai seulement réalisé la `dev`.

### CI/CD

J'ai développé une CI permettant depuis le repo hébergeant le back-end de :

- Tester le code
- Builder l'image Docker contenant le code hébergeant l'API
- Push cet image sur le repository du projet GitLab
- Trigger la CI du repo d'infrastructure en fournissant l'ID de l'image précédemment créée.

Pour cela, il m'a fallu containeriser l'application en passant par l'édition d'un `Dockerfile`.

*Le code est consultable [ici](https://gitlab.com/sdv-open-course-factory/ocf-core/-/blob/develop/.gitlab-ci.yml)*

De même, côté infra la CI permet tout simplement de `plan` et `apply` la configuration Terraform. A terme, les images
du front et backend seront fournies lors du trigger de cette CI depuis les différents repo des développeurs.

*Le code est consultable [ici](https://gitlab.com/sdv-open-course-factory/ocf-core-infra/-/blob/main/.gitlab-ci.yml)*

L'idée derrière ce POC étant de permettre à un développeur de push ses modifications du code back ou front, et d'ainsi
automatiquement déployer l'applicatif sur le cluster Kubernetes.

## Finalement

Cette semaine fût assez intense, mais m'as permis de réaliser un début de lab sur des technologies que je n'ai pas encore
eu l'occasion de tâter lors de mes différentes missions en entreprise. J'ai d'ailleurs prévu de continuer ce POC afin
d'en faire un article plus détaillé sur `AKS` et la solution de CI/CD proposée par GitLab.








