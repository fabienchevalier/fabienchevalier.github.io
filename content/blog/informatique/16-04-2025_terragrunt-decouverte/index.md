---
title: Terragrunt et découverte de l'IAC en mode DRY
categories:
- iac
- terraform
date: "2025-04-16"
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
- Déployer une application web (front + back), dans mon exemple containérisée
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
La logique d'organisation des dossiers est directement inspirée de l'excellent [repo maintenu par Padok](https://github.com/padok-team/docs-terraform-guidelines/blob/main/terragrunt/context_pattern.md), fournissant des bonnes pratiques sur l'utilisation de Terraform, et notamment le concept de `layers`. Je t'invites à le consulter.
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

Au sein du dossier `layers`, dans le repo `marketplace-infrastructure` à la racine, je vais créer un fichier `root.hcl` ([le fichier de configuration Terragrunt `racine`](https://github.com/padok-team/docs-terraform-guidelines/blob/main/terragrunt/context_pattern.md)).

Terragrunt fonctionne de manière récursive. Si tu appliques la configuration Terragrunt du dossier `layers/load-balancer/dev/terragrunt.hcl`, il va remonter dans l'arborescence jusqu'à trouver le fichier `root.hcl` et l'injecter dans la configuration. Le nommage est ici est complètement arbitraire, mais sache que Terragrunt à besoin d'au moins un fichier appelé `terragrunt.hcl` à la racine de chacune de tes stacks.

{{< alert "circle-info" >}}
Le concept de `stack` dans Terragrunt représente une unité de déploiement. Chaque `stack` correspond à un environnement spécifique (par exemple, dev, staging, prod) et peut contenir plusieurs modules ou ressources. En d'autres termes, une `stack` est une collection de ressources Terraform qui sont gérées ensemble.
{{< /alert >}}

#### Fonctions Terragrunt

Terragrunt propose un certain nombre de fonctions permettant de manipuler les fichiers de configuration, de la même manière que les fonctions natives Terraform. J'en utilise quelques unes dans cet article. Sans aller dans le détail, Terragrunt propose des fonctions supplémentaires, plus globales permettant de récupérer du contexte de façon dynamique. Des exemples sont donnés et expliqués dans les sections suivantes. Tu peux retrouver l'ensemble de ces fonctions documentées ici : [Terragrunt Built-in Functions](https://terragrunt.gruntwork.io/docs/reference/built-in-functions/).

#### Configuration des `locals`

Dans `root.hcl`, on va définir définir des [locals](https://developer.hashicorp.com/terraform/language/values/locals), c'est à dire des expressions communes à l'ensemble de mes environnements dans un contexte Terraform. On y retrouvera donc l'`id` du projet, la region, et autres. Voici un exemple :

```hcl
# root.hcl
locals {
  env_name     = basename(get_original_terragrunt_dir())
  region       = "europe-west2"
  env_config   = lookup(local.env_settings, local.env_config, {})
  env_settings = {
    dev = {                            
      project_id = "marketplace-dev"
      vpc        = "marketplace"
      labels     = {
        environment = "dev"
        application = "marketplace"
        managed_by  = "terraform"
      }
    },
    staging = {                            
      project_id = "marketplace-staging"
      vpc        = "marketplace"
      labels     = {
        environment = "staging"
        application = "marketplace"
        managed_by  = "terraform"
      }
    },     
    preprod = {                            
      project_id = "marketplace-preprod"
      vpc        = "marketplace"
      labels     = {
        environment = "preprod"
        application = "marketplace"
        managed_by  = "terraform"
      }
    },
    prod = {                            
      project_id = "marketplace-prod"
      vpc        = "marketplace"
      labels     = {
        environment = "prod"
        application = "marketplace"
        managed_by  = "terraform"
      }      
    }
  }
}

inputs = {}
```

{{< alert "circle-info" >}}
Dans le cas ou l'on souhaite utiliser un Cloud Provider different, comme `AWS` par exemple, il faudra adapter la configuration des `locals` en fonction de la logique de nommage de ton provider. Par exemple, pour AWS, il faudra adapter le `region` et le `project_id` en fonction de la logique de nommage AWS (`account_id` par exemple au lieu du `project_id`).
{{< /alert >}}

{{< alert >}}
Ce fichier de configuration part du principe qu'un réseau VPC a déjà été créé par projet. Ici, dans chaque environnement, on connectera les `assets` au réseau `marketplace`.
{{< /alert >}}

Détaillons un peu ce fichier :

- `env_name`: ici, la fonction `basename()` permet d'extraire le dernier segment d'un chemin donné. Utilisée de pair avec `get_original_terragrunt_dir()`, elle permet d'obtenir le nom de l'environnement actuel (par exemple, dev, staging, etc.) en se basant sur le chemin du répertoire où se situe `terragrunt.hcl`, toujours de manière **absolue**.

- `region`: très simple, ici la région est `hardcodée` car ne changera pas.

- `env_config` : utilise `lookup()` (fonction Terraform) pour récupérer la configuration de l’environnement actuel. Si l’environnement n’existe pas dans `env_settings`, la valeur par défaut `{}` est utilisée pour éviter les erreurs.

- `env_settings`: ce bloc définit une `map` contenant les configurations spécifiques à chaque environnement (par exemple, dev, staging, etc.). Ici, on l'utilise pour spécifier de manière globale le `project_id` et le `vpc` pour chaque environnement (dans l'exemple donné). On pourrait y ajouter d'autres configurations que l'on souhaite centraliser tel que des `tags` etc.

{{< alert "circle-info" >}}
Les fonctions `lookup()` et `basename()` sont des fonctions natives de Terraform. Terragrunt n'étant qu'une surcouche de Terraform, il est possible d'utiliser ces fonctions dans les fichiers de configuration Terragrunt.
{{< /alert >}}

Au final, malgré la complexité apparente, ce fichier `root.hcl` ne fais que fournir des variables génériques de façon dynamique.

#### Gestion du `backend` Terraform

A la suite de la déclaration des `locals`, on va pouvoir définir la configuration du `backend` Terraform. En effet, Terragrunt permet de gérer le `backend` de manière centralisée, et de le générer automatiquement dans chaque `layer`. Voici un exemple de configuration :

```hcl
remote_state {
  backend = "gcs" # Google Cloud Storage, mais peut être S3, etc.
  config = {
    encrypt  = true
    bucket   = "${local.project_id}-tfstates"
    project  = local.project_id
    prefix   = "tfstate/${path_relative_to_include()}"
    location = local.region
  }
  generate = {
    path      = "backend.tf"
    if_exists = "skip"
  }
}
```

{{< alert "circle-info" >}}
On va donc se retrouver avec un fichier `state` par environnement, et par module. Par exemple, pour le module `cloud-run`, on va se retrouver avec un fichier `tfstate` dans le bucket `marketplace-dev-tfstates` (pour l'environnement dev), avec le prefix `tfstate/cloud-run/marketplace-frontend`.
{{< /alert >}}

Terragrunt propose aussi de générer le fichier `provider.tf`, de la même manière que le fichier `backend.tf`. Dans mon exemple, je préfère gérer la configuration du provider dans chaque module, mais sache cela reste une possibilité.

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

{{< alert "circle-info" >}}
Dans l'[exemple fourni par Gruntwork](https://github.com/gruntwork-io/terragrunt-infrastructure-live-example/blob/main/prod/us-east-1/prod/mysql/terragrunt.hcl), tout est centralisé au sein d'un fichier unique `terragrunt.hcl`. En créant un fichier `inputs`, je me permet de séparer la logique de configuration de la logique d'`include`. C'est un choix arbitraire, mais qui me semble plus clair. Même chose pour le fichier `module.hcl`, qui source le module Terraform.
{{< /alert >}}

Commençons par le contenu du fichier `module.hcl`. Rappelles toi, j'ai indiqué en intro qu'il était nécessaire de créer un repo `infrastructure-shared-modules` pour y stocker mes modules. C'est ici que je vais les référencer :

```hcl
# module.hcl
terraform {
  source = "git@github.com:my-org/infrastructure-shared-modules.git//modules/cloudrun?ref=cloudrun-v1.0.0"
}
```

{{< alert >}}
Ici, je fais un choix. Je pourrais sourcer directement le module dans chaque `terragrunt.hcl`, afin d'être en mesure d'avoir un module pour chaque environnement. Cela-dit, dans une logique qui se veut `ISO`, je préfères que chaque environnements soient identiques à la production.
{{< /alert >}}

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
    A --> C[module.hcl]
    A --> D[inputs.hcl]
{{< /mermaid >}}

Terragrunt parcourra depuis le dossier courant, et remontera dans l'arborescence jusqu'à trouver les fichiers `root.hcl`; `module.hcl` et `inputs.hcl`. Il va ensuite fusionner le contenu de ces fichiers, en appliquant la stratégie de fusion définie dans chaque `include`. Dans notre cas, on a choisi la stratégie `deep`, qui va permettre de fusionner les objets imbriqués. La [documentation de Terragrunt](https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#include) permettra d'en savoir plus sur les différentes stratégies de fusion, et le fonctionnement d'include en général.

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

{{< alert >}}
Il est préférable, dans un contexte ou des dépendances sont présentes, de passer par `run-all` afin que Terragrunt puisse correctement mettre à jour l'état des ressources. En effet, si tu appliques les configurations une par une, certaines valeurs peuvent être obsolètes (ex. `connection_string` d'une base de données), et donc entraîner des erreurs lors de l'application des configurations.
{{< /alert >}}

## Last but not least : la gestion des dépendances

Tu t'es peut être posé la question à le lecture de cet article : si Terragrunt cherche à appliquer une configuration dépendante d'une autre, comment fait-il pour savoir dans quel ordre appliquer les ressources ? En effet, si je souhaite créer un `cloud-run` et lui permettre de se connecter à une base de données, il faut que la base de données soit créée avant le `cloud-run`. Terragrunt propose une fonctionnalité permettant de gérer les dépendances entre modules, et donc d'appliquer les ressources dans le bon ordre.

{{< alert "circle-info" >}}
Attention, Terragrunt reste un **wrapper** Terraform. Pour que le système de dépendances fonctionne, il faut que tes modules Terraform puissent `output` des valeurs que tu pourras ensuite utiliser dans d'autres modules. Par exemple, si tu souhaites créer un `cloud-run` qui se connecte à une base de données, il faut que le module de la base de données `output` la `connection_string` de la base de données, afin que le module `cloud-run` puisse s'y connecter.
{{< /alert >}}

Concrètement, cela peut donner quelque chose comme ça pour un `cloud-run` ayant besoin d'exposer une variable d'environnement `DATABASE_URL` :

```hcl
# cloud-run/inputs.hcl
dependency "cloud_sql" {
  config_path = "cloud-sql/${basename(get_original_terragrunt_dir())}"
  mock_outputs = {
    connection_string = "postgres://user:password@host:port/dbname"
  }
}

locals {
  # On récupère la configuration commune définie dans root.hcl
  common = read_terragrunt_config(find_in_parent_folders("root.hcl")).locals
  config = local.common.config
}

inputs = {
  project_id = local.config.project_id
  region     = local.common.region
  env_name   = local.common.env_name
  
  secrets = {
    DATABASE_URL = {
      value = dependency.cloud_sql.outputs.connection_string
    }
  }
}
```

Attardons nous un peu sur la partie `mock_output`. C'est une des fonctionnalités Terragrunt qui peuvent se révéler intéressantes pour le développement. En effet, si tu souhaites tester ta configuration sans avoir à créer l'ensemble des ressources, tu peux utiliser `mock_output` pour simuler les valeurs de sortie d'un module. Cela te permet de `plan` sans obtenir d'erreur lorsque le module n'est pas encore créé.

## Pour conclure

Concrètement, la difficulté se situe vraiment dans la compréhension de la logique de `merge` de Terragrunt. Pour maintenir une infrastructure `DRY`, Terragrunt utilise des fonctions et des `includes` pour fusionner les fichiers de configuration à l'`apply`. Cela nécessite donc (comme souvent lorsqu'on automatise) de rendre dynamique la configuration, et de recourir à des fonctions comme `get_parent_terragrunt_dir()` ou `get_original_terragrunt_dir()` afin de variabiliser un maximum.

Tu l'as bien compris, la mise en place d'une architecture `DRY` avec Terragrunt reste relativement complexe de part l'abstraction qu'elle propose. Selon moi, la valeur ajoutée vaut le coup si :

- Ton projet possède de nombreux environnements à gérer (dev, staging, prod, etc..).
- Tu dois gérer un grand nombre d'applications/infrastructures identiques, mais dont la configuration diffère par environnement (une BDD plus petite en dev...).
- Tu souhaites gérer des modules Terraform de manière centralisée, et versionnée, et les appliquer à l'ensemble de tes environnements.

Terragrunt devrais être un gros `no-go` si :

- Ton projet est de petite taille, et que tu n'as pas besoin de gérer plus de deux environnements.
- Tu n'as pas forcément la bande passante et/ou une équipe qui te permettra de maintenir tes modules.
- Tu n'est pas 100% à l'aise avec Terraform : pour débugger du Terragrunt il faut être en mesure de réellement comprendre ce qu'il applique en arrière plan.

Cet article décris l'organisation et la manière de gérer l'infrastructure via les commandes ad-hoc Terragrunt (`terragrunt plan/apply`) afin d'en cerner le fonctionnement. Bien entendu, il est possible d'utiliser Terragrunt dans un pipeline CI/CD. Une [action managée GitHub](https://github.com/gruntwork-io/terragrunt-action) et d'ailleurs proposée par Gruntwork, l'éditeur de Terragrunt.

N'hésites pas à me contacter ou de laisser un commentaire si tu souhaites échanger sur le sujet, j'ai l'impression que Terragrunt commence à avoir le vent en poupe, et je serais ravi d'en discuter avec toi.

## Sources

- [Terragrunt Documentation](https://terragrunt.gruntwork.io/docs/)
- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [terragrunt-infrastructure-live-example](https://github.com/gruntwork-io/terragrunt-infrastructure-live-example/tree/main)
- [docs-terraform-guidelines](https://github.com/padok-team/docs-terraform-guidelines/tree/main)
- [Reduce redundancy in your Terraform code with Terragrunt](https://cloud.theodo.com/en/blog/terraform-code-terragrunt)