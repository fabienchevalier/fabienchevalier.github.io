---
title: Terragrunt et découverte de l'IAC en mode DRY
categories:
- iac
- terraform
date: "2025-03-27"
description: Découverte de Terragrunt et de l'IAC en mode DRY
summary: Découverte du wrapper Terragrunt, permettant de gérer des configurations Terraform en adoptant une approche DRY (Don't Repeat Yourself).
slug: terragrunt-terraform-dry
tags:
- cloud
- terraform
- iac
- terragrunt
showPagination:
  article: True

showTableOfContents:
  article: True

showComments: True

draft: False
---
{{< badge >}}
Nouvel article!
{{< /badge >}}

{{< lead >}}
Terragrunt est un wrapper (surcouche) pour Terraform, conçue pour simplifier et optimiser la gestion des configurations d'infrastructure en suivant le principe DRY (Don't Repeat Yourself). Dans cet article, je vais tenter d'expliquer le fonctionnement de Terragrunt à travers un cas client fictif.
{{< /lead >}}

## Introduction

Reprenant les bonnes pratiques issues du monde du développement, Terragrunt permet de réduire la duplication de code en factorisant les configurations Terraform. Cet article à pour but d'expliquer la mise en place d'un projet Terraform avec Terragrunt, et surtout de comprendre là où Terragrunt apporte de la valeur par rapport à Terraform seul. Attention, Terragrunt nécessite des bases **solides** en configurations Terraform, cet article se destine donc à un public ayant déjà pratiqué Terraform sur des infrastructures moyennes à grandes, et souhaitant en optimiser la gestion afin de passer à l'échelle. Si ce n'est pas ton cas, je t'invite à consulter [cette section](https://blog.stephane-robert.info/docs/infra-as-code/provisionnement/terraform/introduction/) du blog de Stéphane Robert pour te familiariser avec Terraform. Tout y est très bien expliqué.

## Contexte

J'ai été amené à travailler pour un client ayant une infrastructure cloud conséquente, et logiquement beaucoup de ressources à gérer. La principale problématique rencontrée dans la gestion d'une architecture de cette taille avec Terraform est pour moi l'organisation même du code, et la dette technique qui en découle. En effet, à force de multiplier les ressources, variables, environnements etc.. , on peut vite se retrouver avec du code difficile à comprendre et maintenir, surtout lors de phases de build avec des deadlines serrées.

Terragrunt se veut être une réponse efficace à ces problématiques en apportant une organisation **claire** et **modulaire** au code Terraform. En externalisant les paramètres spécifiques à chaque environnement et en automatisant la gestion des dépendances entre modules, Terragrunt offre une solution structurée pour minimiser la dette technique.

![plan-b-jeu](imgs/illustrations1-terraform.jpg "Illustration de circonstance : Plan B est un jeu proposant de ... Terraformer une planète. Cool, non ?")

Cependant, son adoption n'est pas forcément pertinente pour des projets de petite taille, et peut même être contre-productive si mal utilisée *(👋 [Kubernetes](https://www.appvia.io/blog/5-reasons-you-should-not-use-kubernetes))*. De plus, la courbe d'apprentissage peut être assez raide, j'en ai fais les frais.

Comprendre les concepts de modularité, d’héritage de configurations ou encore de gestion des dépendances demande un investissement initial non négligeable. Cela dit, une fois compris et bien utilisé, Terragrunt peut vite se révéler indispensable. Il permet de structurer efficacement des projets complexes, et de réduire la duplication de code (factorisation) de manière élégante.

Dans cet article, je vais donc tenter de t'expliquer comment démarrer un projet en me basant sur l'expérience que j'ai pu acquérir sur des infrastructures de grosses tailles.

Les exemples que je vais fournir se basent sur des configurations GCP, mais la logique est applicable à n'importe quel cloud provider. Il ne s'agira pas ici d'expliquer comment bootstrap tel ou tel ressources, mais plutôt de te montrer comment organiser ton code Terraform avec Terragrunt.

## C'est quoi concrètement, Terragrunt

![schema-terragrunt](imgs/key-features-terraform-code-dry.png "Schéma représentatif d'une configuration Terragrunt, issu du site de Terragrunt")

Terragrunt est avant-tout un outil `CLI`, reprenant la syntaxe de Terraform dans son fonctionnement ad-hoc. On retrouvera donc les fameux `terragrunt plan`, `terragrunt apply`, etc.. Là où ça devient intéressant, c'est que Terragrunt propose en sus des fonctionnalités comme la génération dynamiques de fichiers `backend.tf` (oui, il permet aussi de créer le `bucket` à la volée si celui-ci n'existe pas), des [fonctions](https://terragrunt.gruntwork.io/docs/reference/built-in-functions/) permettant de manipuler des fichiers `HCL` (comme `read_terragrunt_config`), ou encore une gestion avancée des dépendances entre modules. Mais trêve de bavardages, la meilleure façon de comprendre Terragrunt est d'étudier un exemple concret.

## L'organisation du code

L'organisation du code, ainsi que l'architecture des dossiers sont **extrêmement** importante. Encore une fois, l'usage de Terragrunt permet de forcer des bonnes pratiques, mais un mauvais choix d'organisation peut vite se révéler cauchemardesque, en particulier avec Terragrunt. Pour cet article, je vais donc choisir de baser mes explications sur le cas d'un client fictif, mais qui se rapproche de ce que j'ai pu voir dans le monde réel.

### Contexte client

Partons du principe suivant : je souhaite héberger pour le compte d'un client une application [3 tiers](https://fr.wikipedia.org/wiki/Architecture_trois_tiers) classique, afin d'héberger une marketplace. Niveau ressources cloud, on aura donc besoin de :

{{< alert "circle-info" >}}
Je travaille beaucoup sur GCP en ce moment, pour simplifier la rédaction de mon article, je me base donc sur des assets GCP.
{{< /alert >}}

- Configurer un réseau VPC (subneting, firewall rules, etc..).
- Mettre en place une base de données
- Déployer une application web (front + back), dans mon example containérisée
- Mettre en place un load balancer et un orchestrateur de conteneurs, par facilitée je vais ici utiliser CloudRun.

On parle d'IaC, il va donc falloir créer et organiser son code via des repos Git. Les développeurs vont travailler sur leurs repos respectifs, à savoir `marketplace-frontend` et `marketplace-backend`. En parallèle, il va falloir créer un repo appelé `marketplace-infrastructure` permettant de définir l'infrastructure permettant d'héberger l'application (base de données, load balancer, etc..). Enfin, un repo `infrastructure-shared-modules` sera créé afin de gérer et versionner nos modules. Je ne vais pas m'attarder sur les questions de CI/CD et autres, ce n'est pas le sujet ici.

{{< alert  >}}
J'ai volontairement simplifié l'architecture pour l'exemple, en omettant toute la configuration IAM, Landing Zone et autres. Ici, je me focus uniquement sur la configuration Terragrunt.
{{< /alert >}}

### Schéma d'architecture

Schématiquement, on se retrouve avec quelque chose de classique :

![scheme](./imgs/scheme.png "Schéma de l'architecture de l'application")

D'un premier abord, dans le cas où mon client fictif ne souhaite utiliser que deux environnements de développement, pour ce projet unique, Terragrunt serais overkill. Imaginons maintenant que ce même client souhaite 4 environnements, à savoir `dev`, `staging`, `preprod` et `prod`. De plus, il viens de recevoir une demande du métier, nécessitant la mise en place de 6 applications similaires. Tu vois ou je veux en venir ?

### Hiérarchie des dossiers

{{< alert "circle-info" >}}
La logique d'organisation des dossiers est directement inspirée de l'excellent [repo maintenu par Padok](https://github.com/padok-team/docs-terraform-guidelines/tree/main), fournissant des bonnes pratiques sur l'utilisation de Terraform, et notamment le concept de `layers`. Je t'invites à le consulter.
{{< /alert >}}

Je sais donc que je dois livrer la marketplace en premier lieu, sur 4 environnements, mais avec en tête le fait que quelques semaines plus tard la demande évoluera. Là, on rentre dans un cas ou Terragrunt pourra m'être utile, car je vais pouvoir **factoriser** dès le début. Commençons d'abord par l'arborescence type des dossiers dans mon repo d'infrastructure :

```plaintext
└── layers
    ├── certificates
    │   ├── dev
    │   ├── staging
    │   ├── preprod
    │   ├── prod
    ├── cloud-armor
    │   ├── web-backend
    │   │   ├── dev
    │   │   ├── staging
    │   │   ├── preprod
    │   │   ├── prod
    ├── cloud-run
    │   ├── marketplace-frontend  
    |   |   ├── dev  
    |   |   ├── staging  
    |   |   ├── preprod  
    |   |   └── prod  
    |   ├── marketplace-backend  
    |        ├── dev  
    |        ├── staging  
    |        ├── preprod  
    |        └── prod  
    ├── sql-database
    │   ├── dev
    │   ├── staging
    │   ├── preprod
    │   └── prod
    ├── secrets
    │   ├── dev
    │   ├── staging
    │   ├── preprod
    │   └── prod
    └── load-balancer
        ├── dev
        ├── staging
        ├── preprod
        └── prod
```

{{< alert "circle-info" >}}
Il manque quelques briques comme la gestion DNS, VPC et autres, mais je ne vais pas m'attarder là-dessus. Je vais me concentrer sur la partie Terragrunt.
{{< /alert >}}

Au sein du dossier `layers`  on se retrouve donc avec un dossier par type d'asset. Au sein de chaque dossier, on va retrouver un dossier par environnement. Jusqu'ici, c'est classique, on pourrais d'ailleurs utiliser cette logique avec du Terraform vanilla. Attardons nous maintenant sur le contenu de ces dossiers.

### Fichiers de configurations Terragrunt

Au sein du dossier `layers`, à la racine, je vais créer deux fichiers : `common.hcl`, ainsi que `root.hcl`.

#### `common.hcl`

Ce fichier aura pour but de définir les `locals`, c'est à dire les variables communes à l'ensemble de mes environnements. On y retrouvera donc l'`id` du projet, la region, et autres. Sans plus de suspense, le voici :

```hcl
# common.hcl
locals {
  root_dir    = get_parent_terragrunt_dir()
  layers_path = "${local.root_dir}/../layers"
  environment = basename(get_original_terragrunt_dir())
  region      = "europe-west3"
  config      = lookup(local.config_by_environment, local.environment, {})
  config_by_environment = {
    dev = {                            
      project_id = "marketplace-dev"
      network    = "marketplace"
    },
    staging = {                            
      project_id = "marketplace-staging" 
      network    = "marketplace"              
    },
    preprod = {
      project_id = "marketplace-preprod"
      network    = "marketplace"
    },
    prod = {                            
      project_id = "marketplace-prod" 
      network    = "marketplace"              
    }
  }
}

inputs = {}
```

Détaillons un peu ce fichier :

- `root_dir` : permet de remonter dans l'arborescence des dossiers, et de pointer vers le dossier `layers`. Je t'invites à consulter la [documentation de la fonction Terragrunt](https://terragrunt.gruntwork.io/docs/reference/built-in-functions/#get_parent_terragrunt_dir) `get_parent_terragrunt_dir()`.

Cette variable me permet d'introduire le concept de fonctions Terragrunt, qui peuvent se reveler très puissantes. Elles permettent de rendre le code `HCL` idempotent. Je m'explique. Le fichier `common.hcl` ici est un fichier central, rassemblant des informations générales que l'on serais amener à fournir dans chaque dossier d'environnement dans le cas ou l'on serais sur du Terraform vanilla. Ici, on va pouvoir factoriser ces informations, et les injecter dans chaque layers. Mais comment ça marche ? 

#### Terragrunt et le concept de hiérarchie

Avant d'aller plus loin, il faut que j'explique la manière dont ont effectue un `apply` avec Terragrunt. En effet, Terragrunt va chercher à appliquer les configurations de manière hiérarchique. Par exemple, si je me place dans le dossier `layers/cloud-run/marketplace-frontend/dev`, et que je fais un `terragrunt apply`, Terragrunt va chercher à appliquer la configuration du dossier courant, mais aussi celle des dossiers parents. Il va donc remonter l'arborescence jusqu'à trouver un fichier `terragrunt.hcl`. Dans notre cas, il va donc remonter jusqu'au dossier `layers`, et appliquer la configuration du fichier `common.hcl`.