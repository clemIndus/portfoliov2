---
title: Conception d’une Infrastructure Réseau Complète Virtualisée
publishDate: 2025-05-02 00:00:00
img: /assets/logo-infra.png
img_alt: Conception d’une Infrastructure Réseau Complète Virtualisée
description: |
  Conception d’une Infrastructure Réseau Complète Virtualisée
tags:
  - Réseaux
---

Dans le cadre de mon BTS SIO SISR, j’ai conçu et déployé une infrastructure réseau complète virtualisée à l’aide d’OpenNebula. Ce projet m’a permis de mettre en place un environnement professionnel simulé, intégrant la haute disponibilité, la sécurité réseau, et la gestion de services essentiels.

L’objectif était de créer une architecture capable d’héberger deux projets majeurs : la redondance d’un Active Directory (AD) et la redondance de firewalls PfSense. L’infrastructure repose sur un découpage réseau structuré en quatre VLAN (WAN, LAN, DMZ, CLIENT), assurant une isolation logique des différents services.

<h5>Composants de l’infrastructure :</h5>
<ul>
  <li>2 contrôleurs de domaine (AD) avec services DNS/DHCP</li>

  <li>1 machine cliente Windows</li>
  <li>1 serveur de fichiers</li>
  <li>1 serveur de sauvegarde</li>
  <li>2 firewalls PfSense en redondance</li>
  <li>1 reverse proxy HAProxy avec 2 serveurs web en backend</li>
  <li>1 serveur Zabbix pour la supervision</li>
  <li>1 instance GLPI pour la gestion de parc et des tickets</li>
</ul>

Toutes les machines ont été déployées à partir de templates (Ubuntu / Windows Server) afin de garantir un déploiement reproductible.

<h5>Compétences mobilisées :</h5>

<ul>
  <li>Configuration réseau avancée (VLAN, routage, NAT, HA)</li>

  <li>Administration Windows Server (AD, DNS, DHCP)</li>

  <li>Sécurité réseau (PfSense, segmentation)</li>

  <li>Déploiement et supervision de services critiques</li>

  <li>Automatisation via templates dans un environnement cloud privé</li>
</ul>

Ce projet m’a permis de maîtriser l’ensemble des briques d’un système d’information moderne, de la virtualisation jusqu’à la sécurité, tout en renforçant ma capacité à concevoir une infrastructure stable et évolutive.