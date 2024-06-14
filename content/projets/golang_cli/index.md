---
title: "Golang & CobraCLI : réalisation d'une application en ligne de commande"
date: 2024-04-06
description: "Réalisation d'une application CLI en Go"
showComments: True
showTableOfContents:
  article: True

draft: False
---
{{< lead >}}
Réalisé dans le cadre de ma première année de Master, ce projet noté à pour but de concevoir une application en CLI permettant de gérer/configurer le backend IAM [Casdoor](https://casdoor.org/).

L'ensemble du code source est disponible en open-source [sur GitLab](https://gitlab.com/sdv9972401/casdoor-cli)
{{< /lead >}}

## Introduction

Le but de ce projet, outre la découverte de [Golang](https://go.dev/) est de réfléchir à la manière dont une application en CLI est construite. La seule contrainte imposée par le sujet étant le fait d'utiliser Go, et de choisir parmi 3 backends IAM à savoir :

- [Casdoor](https://casdoor.org/)
- [Authentik](https://goauthentik.io/)
- [PocketBase](https://pocketbase.io/)

Le choix des framework utilisés est complètement libre. Pour ma part, j'ai choisi [Cobra](https://github.com/spf13/cobra), de part sa maturité et sa massive adoption dans divers projets d'envergure (comme [Docker CLI](https://github.com/docker/cli)).

{{< alert "circle-info" >}}
J'ai d'ailleurs découvert que [Hugo](https://gohugo.io/), le framework que j'utilise pour construire mon site web utilise Cobra pour sa CLI!
{{< /alert >}}

### Problématique

#### Généralités

Avant de commencer à coder, il faut savoir dans quelle direction on souhaite aller. La première chose à faire étant d'installer son environnement de développement. Ici, ayant choisi Casdoor il à fallu embarquer :

- Un container hébergeant l'application Casdoor
- Une base de donnée MySQL
- Un fichier permettant d'initialiser la base de donnée de Casdoor au boot

Le but étant de permettre au correcteur de facilement tester mon application, l'ensemble se doit d'être facilement vérifiable. Ici, un simple `docker compose up -d` permet de lancer l'ensemble. La documentation du projet se doit d'être claire et agréable à lire. Pour ce faire, j'ai rédigé un [README.md](https://gitlab.com/sdv9972401/casdoor-cli/-/blob/main/README.md?ref_type=heads) détaillant l'ensemble des fonctionnalités incluses ainsi qu'une documentation permettant d'utiliser mon programme.

Mon programme doit en outre être en mesure de :

- Créer/Modifier/Supprimer (CRUD) les utilisateurs sur Casdoor
- Intégrer une gestion des droits (qui peut faire quoi)

#### Architecture du programme

![image](https://gitlab.com/sdv9972401/casdoor-cli/-/raw/main/img/t-rec.gif)

Dans une interface en ligne de commande, chaque commande est exécutée séquentiellement, généralement de manière isolée. Contrairement à une application graphique ou une API où l'état peut être maintenu en permanence en mémoire vive, un programme CLI doit trouver des moyens alternatifs pour sauvegarder les états entre les exécutions de commandes. Dans le cadre de mon projet, il s'agit d'être en mesure de persister certaines données comme un token de login, de façon sécurisée.

Le framework [Cobra](https://github.com/spf13/cobra) utilisé dans mon application propose une façon d'organiser son code en suivant une méthode bien précise, [détaillée dans sa documentation](https://github.com/spf13/cobra/blob/main/site/content/user_guide.md).

### Réalisation

L'architecture de mon `repo` est présentée comme ceci : 

``` bash
├── LICENSE
├── Makefile
├── README.md
├── cmd
│   ├── groups.go
│   ├── login.go
│   ├── oauth.go
│   ├── root.go
│   └── users.go
├── conf.yml
├── config.yaml.example
├── dev
│   └── app.conf
├── docker-compose.yaml
├── go.mod
├── go.sum
├── handlers
├── helpers
│   ├── authorize.go
│   ├── roles.go
│   └── users.go
├── img
│   ├── logo.png
│   ├── screenshoot.png
│   ├── screenshoot_1.png
│   ├── screenshoot_2.png
│   ├── screenshoot_3.png
│   └── t-rec.gif
├── img.png
├── init_data.json
├── logger
│   └── logger.go
├── main.go
├── models
│   └── models.go
└── utils
    ├── colors.go
    ├── keyring.go
    └── table.go
```

#### main.go

A la racine du dossier, on a différent fichiers dont le `main.go` permettant d'initialiser le programme. J'ai séparé mon programme en différents dossier (modules Go) afin de garder une certaine cohérence lors du développement.

#### cmd

Ce dossier regroupe l'ensemble des commandes disponible dans ma `CLI`. Chaque fichier suit la même logique, et se base sur les méthodes détaillées dans la documentation de `Cobra`.

#### helpers

Les `helpers` regroupent des fonctions middlewares, ici permettant de gérer l'authentification `OAuth2` et la gestion des rôles utilisateurs.

#### utils

Ce dossier regroupes quelques utilitaires que j'ai développé, comme la gestion des couleurs dans l'output, et le backend permettant de sauvegarder des tokens de façon sécurisé ([go-keyring](https://github.com/zalando/go-keyring)).

## Conclusion

Pour une revue plus en détail du projet, je t'invites à consulter le repo disponible [ici](https://gitlab.com/sdv9972401/casdoor-cli). Le [README.md](https://gitlab.com/sdv9972401/casdoor-cli/-/blob/main/README.md?ref_type=heads) détaille l'ensemble des procédures nécessaires pour lancer l'outil.

Ce projet fût ma réelle première interaction avec `Golang`. J'y ai découvert un langage très typé, assez simple à appréhender. Je pense en approfondir la maîtrise, car la plupart des outils `DevOps` les plus [courants sont codés en Go](https://www.scaler.com/topics/devops-tutorial/golang-for-devops/).