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

draft: True
---
{{< badge >}}
Nouvel article!
{{< /badge >}}


{{< lead >}}
Terragrunt est un wrapper (surcouche) pour Terraform, conçue pour simplifier et optimiser la gestion des configurations d'infrastructure en suivant le principe DRY (Don't Repeat Yourself). 
{{< /lead >}}

## Introduction

Reprenant les bonnes pratiques issues du monde du développement, Terragrunt permet de réduire la duplication de code en factorisant les configurations Terraform. Cet article à pour but d'expliquer la mise en place d'un projet Terraform avec Terragrunt, et surtout de comprendre là où Terragrunt apporte de la valeur par rapport à Terraform seul. Attention, Terragrunt nécessite des bases **solides** en configurations Terraform, cet article se destine donc à un public ayant déjà pratiqué Terraform sur des infrastructures moyennes à grandes, et souhaitant en optimiser la gestion afin de passer à l'échelle. Si ce n'est pas ton cas, je t'invite à consulter [cette section](https://blog.stephane-robert.info/docs/infra-as-code/provisionnement/terraform/introduction/) du blog de Stéphane Robert pour te familiariser avec Terraform. Tout y est très bien expliqué.

## Contexte

J'ai été amené à travailler pour un client ayant une infrastructure cloud conséquente, et logiquement beaucoup de ressources à gérer. La principale problématique rencontrée dans la gestion d'une architecture de cette taille avec Terraform est pour moi l'organisation même du code, et la dette technique qui en découle. En effet, à force de multiplier les ressources, variables, environnements etc.. , on peut vite se retrouver avec du code difficile à comprendre et maintenir, surtout lors de phases de build avec des deadlines serrées.

Terragrunt se veut être une réponse efficace à ces problématiques en apportant une organisation **claire** et **modulaire** au code Terraform. En externalisant les paramètres spécifiques à chaque environnement et en automatisant la gestion des dépendances entre modules, Terragrunt offre une solution structurée pour minimiser la dette technique.

![image](imgs/illustrations1-terraform.jpg "Illustration de circonstance : Plan B est un jeu proposant de ... Terraformer une planète. Cool, non ?")

Cependant, son adoption n'est pas forcément pertinente pour des projets de petite taille, et peut même être contre-productive si mal utilisée (👋 Kubernetes). De plus, la courbe d'apprentissage peut être assez raide, j'en ai fais les frais.

Comprendre les concepts de modularité, d’héritage de configurations ou encore de gestion des dépendances demande un investissement initial non négligeable. Cela dit, une fois compris et bien utilisé, Terragrunt peut vite se révéler indispensable. Il permet de structurer efficacement des projets complexes, et de réduire la duplication de code (factorisation) de manière élégante.

Dans cet article, je vais donc tenter de t'expliquer comment démarrer un projet selon *mes* bonnes pratiques 