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

## Architecture de la solution

L'infrastructure s'appuie sur trois nœuds Linux Debian :
- **Manager (Debian 1)** : Orchestrateur du cluster Swarm, héberge le service MariaDB et Swarmpit.
- **Worker 1 (Debian 2)** : Nœud de calcul hébergeant des répliques WordPress et Nginx.
- **Worker 2 (Debian 3)** : Nœud de calcul hébergeant des répliques WordPress et Nginx.

### Images officielles utilisées
- **MariaDB** : `mariadb:12.0.1` (Base de données relationnelle)
- **WordPress** : `wordpress:6.8.2-fpm-alpine` (Moteur applicatif PHP-FPM)
- **Nginx** : `nginx:1.28.0-alpine3.21` (Reverse proxy HTTP & gestionnaire FastCGI)

### Réseaux Overlay
- **frontend** : Réseau overlay sécurisé reliant Nginx aux conteneurs WordPress.
- **backend** : Réseau overlay isolé reliant WordPress à la base de données MariaDB.

---

## Prérequis

- 3 machines virtuelles Linux Debian installées et joignables sur le réseau local.
- Accès root ou privilèges `sudo` sur chaque machine.
- Accès Internet pour télécharger les paquets et les images Docker.
- Droits administrateur sur la machine hôte Windows (pour modifier le fichier `hosts`).

---

## Guide de Déploiement Étape-par-Étape

### Étape 1 : Préparation des machines virtuelles Linux

> À exécuter sur **les 3 machines** (`Debian 1`, `Debian 2`, `Debian 3`).

1. **Mettre à jour le système d'exploitation** :
   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```

2. **Installer les paquets prérequis pour Docker** :
   ```bash
   sudo apt-get install -y ca-certificates curl gnupg lsb-release
   ```

3. **Ajouter la clé GPG officielle de Docker** :
   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   ```

4. **Configurer le dépôt officiel de Docker** :
   ```bash
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
     $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

5. **Installer Docker Engine et le plugin Docker Compose** :
   ```bash
   sudo apt-get update
   sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

6. **Ajouter votre utilisateur courant au groupe docker** (évite d'utiliser `sudo` systématiquement) :
   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```

7. **Activer et démarrer le service Docker au démarrage système** :
   ```bash
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

---

### Étape 2 : Initialisation du cluster Docker Swarm

1. **Sur le Manager (`Debian 1`)** :
   Initialiser le cluster Swarm en spécifiant son adresse IP d'écoute :
   ```bash
   docker swarm init --advertise-addr <IP_DEBIAN_1>
   ```
   *Explication : Cette commande initialise le Swarm et affiche la commande exacte contenant le jeton (token) à copier pour joindre les workers.*

2. **Pour réafficher la commande d'adhésion pour les workers si besoin** :
   ```bash
   docker swarm join-token worker
   ```

3. **Sur les deux Workers (`Debian 2` et `Debian 3`)** :
   Coller la commande obtenue sur chaque worker :
   ```bash
   docker swarm join --token <TOKEN_FOURNI> <IP_DEBIAN_1>:2377
   ```

4. **Vérifier le statut du cluster (sur `Debian 1`)** :
   ```bash
   docker node ls
   ```
   *Résultat attendu : 3 nœuds listés avec le statut `Ready`, et `Debian 1` en tant que `Leader`.*

---

### Étape 3 : Configuration du volume partagé (NFS)

Pour permettre la persistance des fichiers WordPress et médias sur l'ensemble des nœuds :

1. **Sur le Manager (`Debian 1`) : Installer et configurer le serveur NFS** :
   ```bash
   sudo apt-get install -y nfs-kernel-server
   sudo mkdir -p /srv/nfs/wp_data
   sudo chown -R 33:33 /srv/nfs/wp_data
   sudo chmod -R 775 /srv/nfs/wp_data
   ```
   *Note : L'UID 33 correspond à l'utilisateur `www-data` sous Debian/Alpine.*

2. **Déclarer l'export NFS** :
   ```bash
   echo "/srv/nfs/wp_data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   sudo exportfs -arv
   sudo systemctl restart nfs-kernel-server
   ```

3. **Sur les Workers (`Debian 2` et `Debian 3`) : Installer le client NFS** :
   ```bash
   sudo apt-get install -y nfs-common
   ```

---

### Étape 4 : Création des réseaux et des secrets Docker

> À exécuter sur le **Manager (`Debian 1`)**.

1. **Créer les secrets Docker pour la base de données** :
   ```bash
   # Création d'un mot de passe fort pour l'administrateur root MariaDB
   echo "SuperRootSecretPass347!" | docker secret create db_root_password -

   # Création du mot de passe pour l'utilisateur applicatif WordPress
   echo "WpUserSecretPass347!" | docker secret create db_password -
   ```
   *Vérifier la création des secrets :*
   ```bash
   docker secret ls
   ```

---

