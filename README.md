# 347-docker-swarm-wordpress - Déploiement WordPress Haute Disponibilité avec Docker Swarm

Ce dépôt contient l'ensemble des fichiers de configuration et la documentation nécessaire pour le déploiement d'une architecture WordPress hautement disponible, résiliente et évolutive sur un cluster **Docker Swarm** de 3 nœuds (1 Manager et 2 Workers).

---

## Sommaire
1. [Architecture de la solution](#architecture-de-la-solution)
2. [Prérequis](#prérequis)
3. [Étape 1 : Préparation des machines virtuelles Linux](#étape-1--préparation-des-machines-virtuelles-linux)
4. [Étape 2 : Initialisation du cluster Docker Swarm](#étape-2--initialisation-du-cluster-docker-swarm)
5. [Étape 3 : Configuration du volume partagé (NFS)](#étape-3--configuration-du-volume-partagé-nfs)
6. [Étape 4 : Création des réseaux et des secrets Docker](#étape-4--création-des-réseaux-et-des-secrets-docker)
7. [Étape 5 : Déploiement de la stack WordPress](#étape-5--déploiement-de-la-stack-wordpress)
8. [Étape 6 : Déploiement de l'interface d'administration Swarmpit](#étape-6--déploiement-de-linterface-dadministration-swarmpit)
9. [Étape 7 : Configuration DNS et accès utilisateur](#étape-7--configuration-dns-et-accès-utilisateur)
10. [Étape 8 : Procédure de suppression totale de la stack et des volumes](#étape-8--procédure-de-suppression-totale-de-la-stack-et-des-volumes)

---

