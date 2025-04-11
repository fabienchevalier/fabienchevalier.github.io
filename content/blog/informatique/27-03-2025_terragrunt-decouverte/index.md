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
- dry
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
Terragrunt est un wrapper (surcouche) pour Terraform, conçu pour simplifier et optimiser la gestion des configurations d'infrastructure en suivant le principe DRY (Don't Repeat Yourself). Dans cet article, je vais tenter d'expliquer le fonctionnement de Terragrunt à travers un cas client fictif.
{{< /lead >}}

{{< alert >}}
Depuis la création du clone `open-source` de Terraform, à savoir [OpenTofu](https://opentofu.org/), Terragrunt utilisera par défaut `OpenTofu` si il est installé sur ta machine. Dans cet article, je me base sur Terraform mais la logique reste la même.
{{< /alert >}}

## Introduction

Reprenant les bonnes pratiques issues du monde du développement, Terragrunt permet de réduire la duplication de code en factorisant les configurations Terraform. Cet article à pour but d'expliquer la mise en place d'un projet Terraform avec Terragrunt, et surtout de comprendre là où Terragrunt apporte de la valeur par rapport à Terraform seul. Attention, Terragrunt nécessite des bases **solides** en configurations Terraform, cet article se destine donc à un public ayant déjà pratiqué Terraform sur des infrastructures moyennes à grandes, et souhaitant en optimiser la gestion afin de passer à l'échelle. Si ce n'est pas ton cas, je t'invite à consulter [cette section](https://blog.stephane-robert.info/docs/infra-as-code/provisionnement/terraform/introduction/) du blog de Stéphane Robert pour te familiariser avec Terraform. Tout y est très bien expliqué.

## Contexte

J'ai été amené à travailler pour un client ayant une infrastructure cloud conséquente, et logiquement beaucoup de ressources à gérer. La principale problématique rencontrée dans la gestion d'une architecture de cette taille avec Terraform est pour moi l'organisation même du code, et la dette technique qui en découle. En effet, à force de multiplier les ressources, variables, environnements etc.. , on peut vite se retrouver avec du code difficile à comprendre et maintenir, surtout lors de phases de build avec des deadlines serrées.

Terragrunt se veut être une réponse efficace à ces problématiques en apportant une organisation **claire** et **modulaire** au code Terraform. En externalisant les paramètres spécifiques à chaque environnement et en automatisant la gestion des dépendances entre modules, Terragrunt offre une solution structurée pour minimiser la dette technique.

![plan-b-jeu](imgs/illustrations1-terraform.jpg "Illustration de circonstance : Plan B est un jeu proposant de ... Terraformer une planète. Cool, non ?")

Cependant, son adoption n'est pas forcément pertinente pour des projets de petite taille, et peut même être contre-productive si mal utilisée *(👋 [Kubernetes](https://www.appvia.io/blog/5-reasons-you-should-not-use-kubernetes))*. De plus, la courbe d'apprentissage peut être assez raide, j'en ai fais les frais.

Comprendre les concepts de modularité, d’héritage de configurations ou encore de gestion des dépendances demande un investissement initial non négligeable. Cela dit, une fois compris et bien utilisé, Terragrunt peut vite se révéler indispensable. Il permet de structurer efficacement des projets complexes, et de réduire la duplication de code (factorisation) de manière élégante.

Dans cet article, je vais donc tenter de t'expliquer une manière d'utiliser Terragrunt, à travers un exemple fictif. Je pense qu'il est plus facile d'intégrer certains concepts appliqués à une situation concrète, plutôt que de se plonger dans la théorie. Après tout, la [doc officielle](https://terragrunt.gruntwork.io/docs/getting-started/quick-start/) est (très) bien écrite 🙃.

{{< alert "circle-info" >}}
Je présente ici une méthode d'utilisation de Terragrunt, et non **LA** méthode. A tes risques et périls 😎.
{{< /alert >}}

Les exemples que je vais fournir se basent sur des configurations GCP, mais la logique est applicable à n'importe quel cloud provider. Il ne s'agira pas ici d'expliquer comment bootstrap tel ou tel ressources, mais plutôt de te montrer comment organiser ton code Terraform avec Terragrunt.

## C'est quoi concrètement, Terragrunt

![schema-terragrunt](imgs/key-features-terraform-code-dry.png "Schéma représentatif d'une configuration Terragrunt, issu du site de Terragrunt")

Terragrunt est avant-tout un outil `CLI`, reprenant la syntaxe de Terraform dans son fonctionnement ad-hoc. On retrouvera donc les fameux `terragrunt plan`, `terragrunt apply`, etc.. Là où ça devient intéressant, c'est que Terragrunt propose en sus des fonctionnalités comme la génération dynamiques de fichiers `backend.tf` (oui, il permet aussi de créer le `bucket` à la volée si celui-ci n'existe pas), des [fonctions](https://terragrunt.gruntwork.io/docs/reference/built-in-functions/) permettant de manipuler des fichiers `HCL` (comme `read_terragrunt_config`), ou encore une gestion avancée des dépendances entre modules.

{{< alert "circle-info" >}}
Je te recommande de lire au moins le début [du quickstart Terragrunt](https://terragrunt.gruntwork.io/docs/getting-started/quick-start/) présent dans la documentation officielle. Cela t'aideras grandement à comprendre les concepts détaillés par la suite dans cet article.
{{< /alert >}}

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
Je ne fournirais pas le code des modules, ni même la configuration complète de l'infrastructure. Je me limite ici uniquement à l'abstraction Terragrunt. J'ai cependant rédigé un article détaillant la mise en place complète d'une infrastructure via Terraform disponible [ici](https://fchevalier.net/projets/projet_etude_master/).
{{< /alert >}}

### Schéma d'architecture

Pour contextualiser un peu tout ça, voici un schéma d'architecture basique représentant les briques à déployer :

![scheme](./imgs/scheme.png "Schéma de l'architecture de l'application")

D'un premier abord, dans le cas où mon client fictif ne souhaite utiliser que deux environnements de développement, pour ce projet unique, Terragrunt serais overkill. Imaginons maintenant que ce même client souhaite 4 environnements, à savoir `dev`, `staging`, `preprod` et `prod`. De plus, il viens de recevoir une demande du métier, nécessitant la mise en place de 6 applications similaires. Tu vois ou je veux en venir ?

### Hiérarchie des dossiers

{{< alert "circle-info" >}}
La logique d'organisation des dossiers est directement inspirée de l'excellent [repo maintenu par Padok](https://github.com/padok-team/docs-terraform-guidelines/tree/main), fournissant des bonnes pratiques sur l'utilisation de Terraform, et notamment le concept de `layers`. Je t'invites à le consulter.
{{< /alert >}}

Je sais donc que je dois livrer la marketplace en premier lieu, sur 4 environnements, mais avec en tête le fait que quelques semaines plus tard la demande évoluera. Là, on rentre dans un cas ou Terragrunt pourra m'être utile, car je vais pouvoir **factoriser** dès le début.

Pour illustrer mon explication, je vais me concentrer sur deux briques de l'architecture, à savoir le `load balancer` et le `cloud run`. Bien sûr, il faudra configurer bien plus de ressources afin de rendre mon architecture fonctionnelle, mais **je cherche ici à expliquer la logique de factorisation induite par Terragrunt**.

#### Le repo `marketplace-infrastructure`

Voici l'architecture de dossier proposée :

```plaintext
└── layers
    ├── cloud-run
    │   ├── marketplace-frontend  
    |   |   ├── dev  
    |   |   ├── staging  
    |   |   ├── preprod  
    |   |   └── prod  
    |   |── marketplace-backend  
    |        ├── dev  
    |        ├── staging  
    |        ├── preprod  
    |        └── prod  
    └── load-balancer
        ├── dev
        ├── staging
        ├── preprod
        └── prod
etc etc...
```

Ma ressource `cloud-run` permettra de déployer les deux services de mon application, à savoir le `frontend` et le `backend`. Je vais donc créer un dossier `cloud-run`, contenant deux sous-dossiers, un pour chaque service. Chaque service contiendra les environnements de développement, staging, preprod et prod.

Le dossier `load-balancer` contiendra lui aussi les environnements de développement, staging, preprod et prod.

#### Le repo `infrastructure-shared-modules`

Le mot-clé factorisation reviens souvent dans cet article. On a parlé plus haut d'un futur besoin pour mon client d'ajouter de nouvelles applications. Au lieu d'avoir à réécrire (ou copier/coller) le code de chaque module, lors de la création d'une nouvelle application, autant tout rassembler au même endroit dans un repo séparé. Je le hiérarchise de cette manière :

```plaintext
└── modules
    ├── cloud-run
    ├── load-balancer
    ├── vpc
etc etc...
```

Ok, nous avons à présent nos deux repo, ainsi qu'une architecture de dossier claire et facile à comprendre. Passons aux fichiers de configuration.

### Fichiers de configurations Terragrunt

Au sein du dossier `layers`, dans le repo `marketplace-infrastructure` à la racine, je vais créer deux fichiers : `common.hcl`, ainsi que `root.hcl`.

Terragrunt fonctionne de manière récursive. Si tu appliques la configuration Terragrunt du dossier `layers/load-balancer/dev/terragrunt.hcl`, il va remonter dans l'arborescence jusqu'à trouver les fichiers `common.hcl` et `root.hcl` et les injecter dans la configuration. Leur nommage est ici est complètement arbitraire, mais sache que Terragrunt à besoin d'au moins un fichier appelé `terragrunt.hcl` à la racine de chacune de tes stacks.

{{< alert "circle-info" >}}
Le concept de `stack` dans Terragrunt représente une unité de déploiement. Chaque `stack` correspond à un environnement spécifique (par exemple, dev, staging, prod) et peut contenir plusieurs modules ou ressources. En d'autres termes, une `stack` est une collection de ressources Terraform qui sont gérées ensemble.
{{< /alert >}}

#### Fonctions Terragrunt

Terragrunt propose un certain nombre de fonctions permettant de manipuler les fichiers de configuration, de la même manière que les fonctions natives Terraform. J'en utilise quelques unes dans cet article. Sans aller dans le détail, Terragrunt propose des fonctions supplémentaires, plus globales permettant de récupérer du contexte de façon dynamique. Des exemples sont donnés et expliqués dans les sections suivantes.

#### `common.hcl`

Ce fichier a pour but de définir des [locals](https://developer.hashicorp.com/terraform/language/values/locals), c'est à dire des expressions communes à l'ensemble de mes environnements dans un contexte Terraform. On y retrouvera donc l'`id` du projet, la region, et autres. En voici un exemple :

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

{{< alert "circle-info" >}}
Dans le cas ou l'on souhaite utiliser un Cloud Provider different, comme `AWS` par exemple, il faudra adapter la configuration des `locals` en fonction de la logique de nommage de ton provider. Par exemple, pour AWS, il faudra adapter le `region` et le `project_id` en fonction de la logique de nommage AWS (`account_id` par exemple au lieu du `project_id`).
{{< /alert >}}

Détaillons un peu ce fichier :

- `root_dir` : utilise `get_parent_terragrunt_dir()` pour obtenir le chemin absolu du répertoire **du fichier parent Terragrunt** (ex. `root.hcl`). Cela évite de hardcoder le chemin racine, crucial en CI/CD où les chemins varient (ex. `/home/runner/work/...`).

- `layers_path`: dans la même veine que la ligne précédente, cette ligne utilise la variable `root_dir` pour reconstruire le chemin absolu vers le dossier `layers` :

```plaintext
├── common.hcl               # Parent : /mon-projet
└── layers                   # Chemin : /mon-projet/layers
    └── cloud-run
```

- `environnement`: on complexifie un peu. Ici, la fonction `basename()` permet d'extraire le dernier segment d'un chemin donné. Utilisée de pair avec `get_original_terragrunt_dir()`, elle permet d'obtenir le nom de l'environnement actuel (par exemple, dev, staging, etc.) en se basant sur le chemin du répertoire où se situe `terragrunt.hcl`, toujours de manière **absolue**.

- `region`: très simple : ici, la région est `hardcodée` car ne changera pas.

- `config` : utilise `lookup()` (fonction Terraform) pour récupérer la configuration de l’environnement actuel. Si l’environnement n’existe pas dans `config_by_environment`, la valeur par défaut `{}` est utilisée pour éviter les erreurs.

- `config_by_environment`: ce bloc définit une `map` contenant les configurations spécifiques à chaque environnement (par exemple, dev, staging, etc.). Ici, on l'utilise pour spécifier de manière globale le `project_id` et le `network` pour chaque environnement (dans l'exemple donner). On pourrait y ajouter d'autres configurations que l'on souhaite centraliser tel que des `tags` etc.

{{< alert "circle-info" >}}
Les fonctions `lookup()` et `basename()` sont des fonctions natives de Terraform. Terragrunt n'étant qu'une surcouche de Terraform, il est possible d'utiliser ces fonctions dans les fichiers de configuration Terragrunt.
{{< /alert >}}

Au final, malgré la complexité apparente, ce fichier `common.hcl` ne fais que fournir des variables génériques de façon dynamique.

#### `root.hcl`

De la même manière que le fichier `common.hcl`, le fichier `root.hcl` va nous permettre de centraliser certaines configurations. Garde en tête que le nommage de ces fichiers est purement arbitraire, mais reflète selon moi bien leur usage.

```hcl
# root.hcl
locals {
  # On récupère la configuration commune définie dans common.hcl
  common = read_terragrunt_config(find_in_parent_folders("common.hcl")).locals
  config = local.common.config
}

remote_state {
  backend = "gcs"
  config = {
    bucket   = "${local.config.project_id}-tfstates"
    project  = local.config.project_id
    prefix   = "tfstate/${path_relative_to_include()}"
    location = local.common.region
  }
  generate = {
    path      = "backend.tf"
    if_exists = "skip"
  }
}
```

Ici, et c'est une des fonctionnalités que j'apprécie particulièrement, on va pouvoir définir la configuration du `backend` de manière centralisée. En effet, Terragrunt va se charger de générer le fichier `backend.tf` pour nous dans chaque `layers`.

{{< alert "circle-info" >}}
On va donc se retrouver avec un fichier `state` par environnement, et par module. Par exemple, pour le module `cloud-run`, on va se retrouver avec un fichier `tfstate` dans le bucket `marketplace-dev-tfstates` (pour l'environnement dev), avec le prefix `tfstate/cloud-run/marketplace-frontend`.
{{< /alert >}}

### Configuration d'un `layer`

Pour le moment donc, j'ai couvert la configuration `racine`, permettant de factoriser à la base de l'arborescence. Comme dit plus haut, c'est de cette manière que Terragrunt permet de construire une infrastructure en mode `DRY`. Descendons maintenant d'un cran dans l'arborescence, et voyons comment configurer un asset `cloud-run` par exemple.

#### Gestion des modules Terraform de façon DRY

Naviguons dans le dossier `layers/cloud-run/`. Son contenu sera le suivant :

```plaintext
└── cloud-run
    ├── dev
    │   ├── inputs.hcl
    │   └── terragrunt.hcl
    ├── module.hcl
    ├── preprod
    │   ├── inputs.hcl
    │   └── terragrunt.hcl
    ├── prod
    │   ├── inputs.hcl
    │   └── terragrunt.hcl
    └── staging
        ├── inputs.hcl
        └── terragrunt.hcl
```

Commençons par le contenu du fichier `module.hcl`. Rappelles toi, j'ai indiqué en intro qu'il était nécessaire de créer un repo `infrastructure-shared-modules` pour y stocker mes modules. C'est ici que je vais les référencer :

```hcl
# module.hcl
terraform {
  source = "git@github.com:my-org/infrastructure-shared-modules.git//modules/cloudrun?ref=cloudrun-v1.0.0"
}
```

Concrètement, mon repo `infrastructure-shared-modules` va contenir l'ensemble de mes modules, versionnés via `git`.

{{< alert >}}
Je te conseilles vivement de `tag` tes modules, afin de pouvoir revenir en arrière si besoin. En effet, si tu fais évoluer un module, et que tu ne tag pas ta version, tu risques d'appliquer les changements à l'ensemble des configurations.
{{< /alert >}}

En backstage, Terragrunt va cloner ce `repo`, et lui fournir les `inputs` définies dans chaque fichiers `inputs.hcl`. On retrouvera d'ailleurs un dossier `.terragrunt-cache`, contenant le repo cloné dans chaque environnement.

#### Que se passe-t-il lorsque je lance un `terragrunt apply` ?

Un petit coup d'oeil sur le fichier `terragrunt.hcl` permet de se rendre compte de toute la logique expliquée précédemment :

```hcl
include "root" {
  path           = find_in_parent_folders("root.hcl")
  merge_strategy = "deep"
}

include "common" {
  path           = find_in_parent_folders("common.hcl")
  merge_strategy = "deep"
}

include "module" {
  path           = find_in_parent_folders("module.hcl")
  merge_strategy = "deep"
}

include "inputs" {
  path           = "inputs.hcl"
  merge_strategy = "deep"
}
```

Ce qui se résume schématiquement :

{{< mermaid >}}
flowchart TD
    A[terragrunt.hcl] --> B[root.hcl]
    A --> C[common.hcl]
    A --> D[module.hcl]
    A --> E[inputs.hcl]
{{< /mermaid >}}

Terragrunt parcourra depuis le dossier courant, et remontera dans l'arborescence jusqu'à trouver les fichiers `root.hcl`, `common.hcl`, `module.hcl` et `inputs.hcl`. Il va ensuite fusionner le contenu de ces fichiers, en appliquant la stratégie de fusion définie dans chaque `include`. Dans notre cas, on a choisi la stratégie `deep`, qui va permettre de fusionner les objets imbriqués. La [documentation de Terragrunt](https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#include) permettra d'en savoir plus sur les différentes stratégies de fusion, et le fonctionnement d'include en général.

{{< alert "circle-info" >}}
Pour faire simple, on peut partir du principe que Terragrunt va fusionner les fichiers de configuration en un seul fichier, en appliquant la stratégie de fusion définie dans chaque `include`. Il va ensuite appliquer cette configuration au module Terraform. Le fichier `input.hcl` contiendra donc l'équivalent d'un fichier `variables.tf` dans un module Terraform classique. Il va permettre de définir les variables d'entrées pour le module, et sera fusionné avec les autres fichiers de configuration.
{{< /alert >}}

#### Le concept de `terragrunt run-all

Cette commande va permettre d'appliquer l'ensemble des configurations de manière récursive, en une seule commande. C'est bien là que toute la puissance de Terragrunt se révèle. 

```bash
terragrunt run-all apply layers/
```

En effet, si tu as bien suivi, cette commande permettra en une fois de créer l'ensemble des ressources de l'application, pour chaque environnement, et d'automatiquement générer la configuration `backend.tf`. 

{{< alert "circle-info" >}}
Pour peu que tu aie les droits sur ton cloud-provider, Terragrunt se charge même de créer le bucket `tfstates` pour toi, si celui-ci n'existe pas. DRY.
{{< /alert >}}

## Last but not least : la gestion des dépendances

Tu t'es peut être posé la question à le lecture de cet article : si Terragrunt cherche à appliquer une configuration dépendante d'une autre, comment fait-il pour savoir dans quel ordre appliquer les ressources ? En effet, si je souhaite créer un `load balancer`

## Pour conclure

Je t'avais prévenu, la courbe d'apprentissage est assez raide. Mais concrètement, la difficulté se situe vraiment dans la compréhension de la logique de `merge` de Terragrunt. Pour maintenir une infrastructure `DRY`, Terragrunt utilise des fonctions et des `includes` pour fusionner les fichiers de configuration à l'`apply`.

Tu l'as bien compris, la mise en place d'une architecture `DRY` avec Terragrunt reste relativement complexe de part l'abstraction qu'elle propose. Selon moi, la valeur ajoutée vaut le coup si :

- Ton projet possède de nombreux environnements à gérer (dev, staging, prod, etc..).
- Tu dois gérer un grand nombre d'applications/infrastructures identiques, mais dont la configuration diffère par environnement.
- Tu souhaites gérer des modules Terraform de manière centralisée, et versionnée, et les appliquer à l'ensemble de tes environnements.

