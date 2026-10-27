# 347-docker-swarm-wordpress

## Étape 1 : Préparation des machines

À faire sur chaque VM :

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release git keepalived nfs-common
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

Compatibilité API pour Swarmpit :
```bash
echo '{"min-api-version": "1.30"}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
sudo systemctl enable docker
```

---

## Étape 2 : Initialisation du cluster Docker Swarm

1. **Sur le Manager** :
   ```bash
   docker swarm init --advertise-addr <IP_MANAGER>
   ```

2. **Sur les Workers** :
   ```bash
   docker swarm join --token <TOKEN_FOURNI> <IP_MANAGER>:2377
   ```

3. **Haute Disponibilité Swarm** :
   Pour que le cluster continue de tourner même si la VM Manager s'arrête, on promeut les 2 workers au rôle de Manager :
   ```bash
   docker node promote <NOM_WORKER_1> <NOM_WORKER_2>
   ```

4. **Label pour MariaDB** :
   ```bash
   docker node update --label-add db=true <NOM_WORKER_1>
   ```

---

## Étape 3 : Haute Disponibilité Réseau 

Pour que l'adresse reste joignable si le Manager tombe, on utilise une IP virtuelle partagée (`<IP_VIRTUELLE_VIP>`).

### Sur le Manager :
Créer `/etc/keepalived/keepalived.conf` :
```text
vrrp_instance VI_1 {
    state MASTER
    interface <INTERFACE_RESEAU>
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 347ETML
    }
    virtual_ipaddress {
        <IP_VIRTUELLE_VIP>/24 dev <INTERFACE_RESEAU>
    }
}
```
Lancer le service :
```bash
sudo systemctl enable --now keepalived
```

Sur **Worker 1** et **Worker 2**, appliquer la même configuration avec `state BACKUP` et des priorités 100 et 99.

---

## Étape 4 : Volume partagé NFS

Pour stocker les fichiers WordPress et médias :

1. **Sur Worker 1 (serveur NFS)** :
   ```bash
   sudo apt-get install -y nfs-kernel-server
   sudo mkdir -p /srv/nfs/wp_data
   sudo chown -R 33:33 /srv/nfs/wp_data
   sudo chmod -R 775 /srv/nfs/wp_data
   echo "/srv/nfs/wp_data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   sudo exportfs -arv
   sudo systemctl restart nfs-kernel-server
   ```

2. **Sur le Manager et Worker 2 (clients)** :
   ```bash
   sudo apt-get install -y nfs-common
   ```

---

## Étape 5 : Secrets Docker

Depuis le Manager :
```bash
echo "SuperRootSecretPass347!" | docker secret create db_root_password -
echo "WpUserSecretPass347!" | docker secret create db_password -
```

---

## Étape 6 : Déploiement de la stack WordPress

1. **Récupérer le projet** :
   ```bash
   git clone https://github.com/Damien-16/347-docker-swarm-wordpress.git
   cd 347-docker-swarm-wordpress
   ```

2. **Déployer la stack** :
   ```bash
   export NFS_SERVER_IP=<IP_WORKER_1>
   docker stack deploy -c docker-compose.yml wp_stack
   ```

3. **Vérifier les services** :
   ```bash
   docker stack services wp_stack
   ```

---

## Étape 7 : Interface Swarmpit

Déploiement en ligne de commande :
```bash
docker run -it --rm \
  --name swarmpit-installer \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  swarmpit/install:1.9
```

L'interface est accessible sur : `http://<IP_VIRTUELLE_VIP>:888`

---

## Étape 8 : Configuration DNS et Démonstration de tolérance de panne

1. **Sur Windows (`C:\Windows\System32\drivers\etc\hosts`)** :
   Ajouter la ligne :
   ```text
   <IP_VIRTUELLE_VIP>   www.my-wordpress.ch
   ```

2. **Accès au site** :
   Ouvrir le navigateur sur **http://www.my-wordpress.ch**.

3. **Test d'arrêt du Manager** :
   Éteindre la VM Manager (`sudo poweroff`).
   L'IP virtuelle bascule automatiquement sur Worker 1 et le site continue de fonctionner sans interruption.

---

## Étape 9 : Suppression de la stack et des volumes

```bash
# 1. Supprimer les stacks
docker stack rm wp_stack
docker stack rm swarmpit

# 2. Supprimer les volumes persistants
docker volume rm wp_stack_db_data wp_stack_wp_data

# 3. Supprimer les secrets
docker secret rm db_password db_root_password

# 4. Nettoyer les données locales NFS (optionnel)
sudo rm -rf /srv/nfs/wp_data/*
```

---

### Utilisation de l'IA
J'ai utilisé de l'IA comme tuteur afin de m'aider avec certains concept/commandes que je connaissais pas encore.

### Conclusion personelle
J'ai trouvé que ce projet a été très intéressant à découvrir et pratiquer, j'ai appris plain de chose.
