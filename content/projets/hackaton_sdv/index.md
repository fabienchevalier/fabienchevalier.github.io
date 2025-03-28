---
title: "Hackaton Sup De Vinci 2023"
date: 2023-07-03
description: "Réaliser l'infrastructure permettant d'héberger une application web (front end/backend)"
showComments: True
showTableOfContents:
  article: True
summary: Réalisé dans le cadre de mon année de Bachelor, le Hackaton à pour but de proposer une solution informatique en une semaine. Un jury juge ensuite la solution la plus convaincante et désigne les vainqueurs.

draft: False
---
{{< lead >}}
Réalisé dans le cadre de mon année de Bachelor, le Hackaton à pour but de proposer une solution informatique en une 
semaine. Un jury juge ensuite la solution la plus convaincante et désigne les vainqueurs.

L'ensemble du projet que j'ai réalisé est disponible en open-source  
[sur GitLab](https://gitlab.com/sdv-open-course-factory)
{{< /lead >}}

{{< alert >}}
Etant encore étudiant et loin d'être un spécialiste K8S, le projet présenté sur cette page comporte forcément de grosses imprécisions et d'énormes
failles de sécurité. Si par hasard un DevOps confirmé tombe sur ces lignes, n'hésites pas à me contacter via la section
commentaire ou via email, j'aurais quelques questions à te poser :wink:
{{< /alert >}}

## Introduction

Lors de ce hackathon, mon équipe à choisi pour sujet `Solution Libre`. Il s'agissait de :

- développer le frontend d'une application à destination de formateurs et de leurs élèves
- développer l'architecture qui permettra in fine d'héberger l'application, son backend et sa base de données
- proposer une CI/CD permettant de mettre en place une livraison et une intégration continue de l'applicatif et son infrastructure

Le back-end étant fourni par l'école, mon travail ici se limitait à son intégration ainsi qu'a celle du front-end.

### Contexte et travail d'équipe

L'ensemble des étudiants de l'école sont réunis en équipe de 10, tout niveaux et spécialités confondues. Toute la 
difficulté étant de coordonner l'ensemble de l'équipe afin de livrer un produit fonctionnel. 

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
Lors de la réalisation, j'ai regroupé tout mes `pods` K8S au sein du même `namespace` par manque de temps lors du 
développement de la partie Terraform. La partie monitoring est aussi inachevée et ne figure pas sur le repo. Cela dit,
il n'est pas exclu que je m'y penche dans le cadre d'un article de blog dans un futur proche.
{{< /alert >}}

En l'état, l'architecture disponible sur le lien GitLab donnée en préambule permet uniquement de requêter l'API du
backend. De plus, comme indiqué sur le schéma la base de donnée utilisée est hébergée dans un `pod`. J'aurais préféré
mettre en place une base de donnée managée mais n'étant pas encore très à l'aise avec Kubernetes sur Azure, j'ai préféré
aller au plus simple. Le but de cet article reste pour moi l'occasion de garder une sorte de documentation sur le projet
réalisé. **A ne pas utiliser en production donc** :wink:.

### Infrastructure As Code

#### Contexte

L'enjeu étant de provisioner tout cela as code, j'ai déployé cette architecture avec Terraform. Le repo contenant
l'infrastructure est construit comme ceci:

```
└── terraform
    ├── aks.tf
    ├── environment
    │   └── dev
    │       └── variables.tfvars
    ├── kubernetes.tf
    ├── main.tf
    ├── modules
    │   └── kubernetes
    │       ├── main.tf
    │       ├── outputs.tf
    │       └── variables.tf
    ├── outputs.tf
    ├── rg.tf
    ├── variables.tf
    └── vpc.tf
```

Cette architecture est dès le départ conçue pour être en mesure de déployer des environments `ISO` dev/pprd/prod.
Cela permet au développeur d'être en mesure (en théorie) de tester son code en amont sur des architectures identiques
avant de déployer l'applicatif en production.

#### Backend et providers

Dans le fichier `main.tf`, en début de code on retrouve la déclaration du `backend`, ainsi que les `providers` requis:

```terraform
terraform {
  backend "azurerm" {
    resource_group_name  = "backend-terraform-rg"
    storage_account_name = "backendhackatonsdv"
    container_name       = "terraform-state"
    key                  = "terraform.tfstate"
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "3.63.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "2.21.1"
    }
  }
}

provider "azurerm" {
  features {}
  skip_provider_registration = true
}

provider "kubernetes" {
  host                   = module.aks.host
  client_certificate     = base64decode(module.aks.client_certificate)
  client_key             = base64decode(module.aks.client_key)
  cluster_ca_certificate = base64decode(module.aks.cluster_ca_certificate)
}

```

Le `backend`, permettant de stocker le fichier `tfstate` consiste en un compte de stockage Azure hébergeant ce même
fichier. Dans le cas de ce lab, j'ai utilisé un compte de stockage préalablement existant sur mon compte Azure.

J'utilise les providers `azure_rm` et `kubernetes` permettant de respectivement :

- intéragir avec mon compte Azure
- intéragir avec le cluster Kubernetes déployé

{{< alert "circle-info" >}}
J'utilise un compte Azure Students fourni par l'école pour ce lab. Terraform refusais 
systématiquement d'`apply` mon infrastructure en raison d'erreurs de droits. L'option
`skip_provider_registration = true` m'as permis de débloquer la situation.
{{< /alert >}}

Le provider `kubernetes` nécessite une configuration permettant de se connecter au cluster afin d'y déployer les
ressources. Ici, je fournis les valeurs dynamiquement à partir du module `aks` que je decris plus loin dans l'article.

#### Terraform et modules

Pour une meilleure lisibilité, je préfère séparer chaque ressources déployées en un fichier distinct. Cela évite de
se retrouver avec un gros fichier de plusieurs centaines de lignes devenant vite illisible. Terraform permet ensuite
d'organiser son code en modules. Un module est un ensemble de fichiers Terraform stockés dans un dossier. Cela 
permet d'éviter les répétitions dans le code et la méthode de fonctionnement peut être comparable à une fonction dans 
un programme informatique. Ici j'ai donc : 

- `aks.tf` -> Crée un cluster Kubernetes managé via le module `Azure/aks/azurerm`
- `kubernetes.tf` -> Intéragit avec le cluster Kubernetes. Ce fichier appelle un module que j'ai développé et stocké dans le répertoire `modules`
-  `rg.tf` -> Le groupe de ressource Azure contenant l'ensemble des instances
- `vpc.tf` -> La configuration réseau de l'infrastructure

Pour finir, dans le fichier `variables.tf`, je déclare les variables qui seront utilisées pour déployer l'infrastructure.
Ces variables sont fournies par le fichier `variables.tfvars`, qui diffère en fonction de l'environement de production
choisi. Ainsi, au `plan` ou `apply` il suffira de rajouter le flag `-var-file=environment/dev/variables.tfvars` afin de
définir l'environment choisi. Dans ce lab, il s'agit de l'environment de `dev`.

### CI/CD

#### Containerisation

Pour être en mesure de déployer l'infrastructure sur Kubernetes, il m'a fallu au préalable containairiser le back-end
dans une image Docker prête à être stocké sur un registre d'images (ici, j'utilise celui de GitLab). La méthode aurais
été similaire pour le front-end. J'ai donc écris un Dockerfile : 

```Dockerfile
# Base Golang Image
FROM golang:latest

# Setup working directory
WORKDIR /usr/src/osf-core

# Copy source code to
COPY . /usr/src/osf-core

# Install Git and NodeJS
RUN curl -sL https://deb.nodesource.com/setup_16.x | bash -
RUN apt-get install -y nodejs npm

# Install NPM dependencies
RUN npm install -g @marp-team/marp-core \
    && npm install -g markdown-it-include \
    && npm install -g markdown-it-container \
    && npm install -g markdown-it-attrs

# Install Go Library & Swagger
RUN cd /usr/src/osf-core && go get golang.org/x/text/transform \
    && go get golang.org/x/text/unicode/norm \
    && go install github.com/swaggo/swag/cmd/swag@v1.8.12

# Init Swagger
RUN cd /usr/src/osf-core && swag init --parseDependency --parseInternal

# Export ports
EXPOSE 8000/tcp
EXPOSE 443/tcp
EXPOSE 80/tcp

# Launch the API
CMD ["go", "run", "/usr/src/osf-core/main.go"]
```
J'ai tenté d'utiliser les fichiers `packages.json` et `go.sum`/`go.mod` afin d'installer les dépendances directement
depuis ces fichiers, mais la génération de mon image plantait. De plus, j'ai du forcer la version de `swagger` en `1.8.12`
car un problème de compatibilité m'empêchais de générer la documentation Swagger.

Ce Dockerfile est utilisé dans la CI afin de générer dynamiquement l'image destiné à être poussée dans le cluster K8S.

#### CI applicative

La CI applicative est très basique pour ce lab mais il y a quelques spécificités. J'utilise une image `docker in docker`,
car pour builder l'image applicative il est nécessaire d'executer des commandes docker dans docker
([plus d'infos ici](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html)). Pour fonctionner correctement, le build
d'une image Docker par le runner nécessite d'ajouter ces lignes en début de CI : 

```yaml
image: docker:20.10.16
services:
  - docker:20.10.16-dind

variables:
  DOCKER_TLS_CERTDIR: "/certs"
```

Sans cela, docker serais incapable d'accéder à Internet à l'intérieur du container.

La CI consiste en 3 stages : 

- test
- deploy
- trigger_deploy_to_terraform

2 variables sont à renseigner manuellement : `ENV` et `TAG`:

```yaml
  ENV:
    value: "dev"
    description: "On wich env the image should be deployed"
  TAG:
    value: "latest"
    description: "Version of the image"
```

Ces variables sont utilisées par la suite pour définir sur quel environment déployer l'image, et permet à Terraform
de récupérer le chemin vers l'image à pousser sur le cluster. Le script qui va créer l'image est très simple : 

```yaml
deploy:
  stage: deploy
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_TOKEN $DOCKER_REGISTRY_URL
    - docker build -t registry.gitlab.com/sdv-open-course-factory/ocf-core/${ENV}-backend:${TAG} .
    - docker push registry.gitlab.com/sdv-open-course-factory/ocf-core/${ENV}-backend:${TAG}

```
Le nom et le chemin de l'image sera renseigné automatiquement en fonction des données entrées par le dev.

Pour finir, le dernier stage déclenche la CI situées sur le repo Terraform, en poussant la variable `TAG` permettant
à Terraform de pousser la bonne image du backend sur le cluster Kubernetes:

```yaml
 New job to trigger the other project's CI
trigger_deploy_to_terraform:
  image: curlimages/curl
  stage: trigger_deploy_to_terraform
  script:
    - curl -X POST --fail -F token=$CI_TRIGGER_TOKEN -F "ref=main" -F "variables[TAG]=$TAG" https://gitlab.com/api/v4/projects/47370418/trigger/pipeline

  needs:
    - deploy


```

{{< alert >}}
Je n'ai pas encore variabilisé l'environement à ce niveau. Pour le moment, ma CI est uniquement en mesure de déployer
sur dev. Encore une fois, question de priorités niveau timing. Je cherchais surtout à avoir quelque chose de fonctionnel
au plus vite. Ainsi, seul la variable `TAG` est envoyée à Terraform, permettant de retrouver l'image.
{{< /alert >}}

#### CI infrastructure

La CI d'infrastructure peut soit être déclenchée par le job `trigger` de la CI applicative, soit manuellement. On retrouve
les stages classiques : 

- validate
- plan
- apply
- destroy

J'utilise ici l'image `hashicorp/terraform:latest`, m'évitant d'installer Terraform à chaque déploiement sur le runner.

Au niveau des variables : 

```yaml
variables:
  TF_ROOT: ${CI_PROJECT_DIR}/terraform/
  TF_ENVIRONMENT: "dev"  # Définissez l'environnement souhaité ici (par exemple, dev, preprod, prod)
  TF_DESTROY:
    description: Destroy Terraform resources
    value: "false"
```
{{< alert "circle-info" >}}
Comme dit précédemment, l'environement et codé en dur sur `dev` dans mon lab
{{< /alert >}}

La variable `TF_DESTROY` me permet de détruire l'infrastructure via la CI. Le stage `destroy` ne s'execute seulement
ci la valeur est changée pour `true`:

```yaml
terraform_destroy:
  stage: destroy
  script: terraform destroy -auto-approve -var-file=environment/${TF_ENVIRONMENT}/variables.tfvars -var="img_tag=${TAG}"
  rules:
    - if: $TF_DESTROY == "true"
      when: always
```

La partie `plan` contient un simple script `bash` qui va tester l'existence de la variable `TAG` récupérée depuis la CI
applicative : 

```bash
if [ -n "$TAG" ]; then
    terraform plan -var-file=environment/${TF_ENVIRONMENT}/variables.tfvars -var "img_tag=${TAG}"
    else
      echo "Nothing to plan"
fi
```

J'utilise le flag `-var-file=environment/${TF_ENVIRONMENT}/variables.tfvars` afin de déterminer dynamiquement sur 
quel environenment déployer. Puis le flag `-var "img_tag=${TAG}` pour déterminer l'image Docker à utiliser sur le même
principe.

Le flag `-var "img_tag=${TAG}` permet de définir la variable Terraform contenue dans le `variables.tf` à la racine ainsi
que dans le module Kubernetes:

```terraform
variable "img_tag" {
  description = "Image tag"
  type        = string
}
```

L'adresse permettant de pointer vers l'image est ensuite reconstruite comme ceci lors de la création du déploiement
Kubernetes:

```terraform
spec {
        container {
          image = "registry.gitlab.com/sdv-open-course-factory/ocf-core/${var.env}-backend:${var.img_tag}"
          name  = "ocf-core-backend"

          port {
            container_port = 80
          }

          port {
            container_port = 443
          }

          port {
            container_port = 8000
          }
```

## Conclusion

J'ai eu au final à peu près 4 jours pour réaliser toute cette architecture. C'est assez court, mais le POC est fonctionnel
et permet d'accéder à la doc `swagger` du backend et de manipuler l'API. Dans l'idée, pour vraiment déployer ce lab en production il faudrais:

- Bien sûr y intégrer un front-end
- Retravailler la CI pour intégrer l'image du front-end
- Intégrer toute la partie monitoring et gestion des logs, c'est super important
- Utiliser une BDD managée Azure au lieu d'une image `PostegrSQL` dans le cluster
- Séparer chaque services sur des namespaces différents
- Niveau sécurité, placer le loadbalancer sur un subnet séparé.
- Mettre en place un bastion sur un VPC différent afin d'être en mesure de requêter l'API K8S via `kubectl`. Ici, :warning: l'API est exposée au web :warning:

Je pense ré-utiliser mon architecture pour tenter de réaliser le front-end. J'en profiterais pour consolider le tout
par rapport à la liste ci-dessus. Cela fais aussi quelque temps que je suis intéressé
par les technologies de dev orienté front-end (`React`) et n'ayant pas ou très peu d'expérience en `JavaScript`, ce serais
l'occasion. Affaire à suivre donc...






