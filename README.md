# Déploiement Automatisé de l'Agent GLPI avec Ansible

## Objectif du Projet
Ce projet automatise l'installation, la configuration et le déploiement de l'agent GLPI sur un parc de serveurs Linux (Debian/Ubuntu). Conçu pour être robuste et reproductible, il gère l'intégralité du cycle de vie : de l'initialisation des accès SSH sécurisés jusqu'à la remontée du premier inventaire matériel et logiciel.

## Architecture de Référence

Le déploiement s'articule autour des nœuds suivants :

* **Serveur Ansible (Nœud de contrôle)** : `DEB-ANSIBLE-1`
* **Serveur GLPI (Destination de l'inventaire)** : `deb-server-2`
* **Machines cibles (Clients Linux)** :
    * `deb-server-1`
    * `deb-server-2`
    * `deb-web-1`
    * `deb-web-2`

## Structure du Code

```text
/opt/ansible
├── ansible.cfg
├── group_vars/
│   └── linux_clients.yml
├── inventory/
│   └── production.ini
├── playbooks/
│   └── deploy-glpi-agent.yml
└── roles/
    └── glpi-agent/
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        └── templates/
            └── agent.cfg.j2
```

## Phase 1 : Préparation de l'Infrastructure (Bootstrap)

Avant l'exécution d'Ansible, les serveurs cibles doivent être préparés pour autoriser les connexions sécurisées par clé SSH et l'exécution de commandes avec élévation de privilèges.

### 1. Création de l'utilisateur de service
Sur chaque machine cible, créez l'utilisateur dédié et ajoutez-le au groupe sudo :
```bash
sudo adduser ansible
sudo usermod -aG sudo ansible
```

### 2. Configuration de l'élévation de privilèges (Sudo)
Pour qu'Ansible puisse installer des paquets sans interruption, configurez le sudo sans mot de passe. Sur chaque machine cible :
```bash
sudo visudo
```
Ajoutez la ligne suivante à la fin du fichier :
```text
ansible ALL=(ALL) NOPASSWD: ALL
```

### 3. Génération de la clé SSH
Sur le **serveur Ansible** (`DEB-ANSIBLE-1`), générez une clé cryptographique forte (Ed25519) :
```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible -C "ansible@server"
```

### 4. Distribution des clés publiques
Depuis le serveur Ansible, injectez la clé publique sur toutes les machines cibles :
```bash
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-server-1
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-server-2
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-web-1
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-web-2
```

Vérifiez ensuite que la connexion s'effectue sans mot de passe : `ssh -i ~/.ssh/ansible ansible@deb-server-1`.

## Phase 2 : Configuration d'Ansible

### Fichier de configuration principal
Fichier : `ansible.cfg`
```ini
[defaults]
roles_path = ./roles
```

### Inventaire
Fichier : `inventory/production.ini`
```ini
[linux_clients]
deb-server-1
deb-server-2
deb-web-1
deb-web-2

[linux_clients:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/ansible
```

### Configuration du Serveur GLPI

L'URL du serveur GLPI (`http://deb-server-2/front/inventory.php`) est définie à deux niveaux pour garantir un comportement prédictible par défaut, tout en permettant une surcharge au niveau du groupe.

**1. Variables par défaut du rôle (Fallback)**
Fichier : `roles/glpi-agent/defaults/main.yml`
```yaml
glpi_server: "http://deb-server-2/front/inventory.php"
glpi_verify_ssl: false
glpi_agent_package_url: ""
glpi_delay: 3600
glpi_tag: "Ansible"
```

**2. Variables de groupe (Surcharge)**
Fichier : `group_vars/linux_clients.yml`
```yaml
glpi_server: "http://deb-server-2/front/inventory.php"
glpi_verify_ssl: false
glpi_agent_version: "latest"
```

## Phase 3 : Déploiement

### 1. Test de connectivité global
Validez que l'infrastructure est prête et qu'Ansible peut élever ses privilèges :
```bash
ansible linux_clients -i inventory/production.ini -m ping --become
```

### 2. Exécution du Playbook
Lancez le déploiement sur l'ensemble du parc :
```bash
ansible-playbook -i inventory/production.ini playbooks/deploy-glpi-agent.yml
```

### Mécanique interne du rôle `glpi-agent`
Lors de l'exécution, le rôle effectue les actions suivantes de manière idempotente :
1. Installation des dépendances système (`curl`, `perl`).
2. Interrogation de l'API GitHub pour récupérer l'URL du dernier installeur de l'agent GLPI.
3. Téléchargement et exécution du script Perl d'installation.
4. Déploiement du template de configuration (`/etc/glpi-agent/agent.cfg`).
5. Activation (enabled) et démarrage (started) du service systemd `glpi-agent`.
6. Déclenchement immédiat de la remontée d'inventaire.

## Sécurisation Post-Déploiement (Recommandé)

Une fois l'automatisation par clé SSH validée, verrouillez l'accès aux serveurs cibles en désactivant l'authentification par mot de passe.

Sur les machines cibles, éditez la configuration SSH :
```bash
sudo nano /etc/ssh/sshd_config
```
Appliquez les paramètres suivants :
```text
PasswordAuthentication no
PermitRootLogin no
```
Redémarrez le service pour appliquer les restrictions :
```bash
sudo systemctl restart ssh
```

## Maintenance et Dépannage

Si un client ne remonte pas dans GLPI, voici les commandes de diagnostic à exécuter sur la machine cible :

* **Vérifier l'état du service** :
    ```bash
    systemctl status glpi-agent
    ```
* **Consulter les logs locaux** :
    ```bash
    journalctl -u glpi-agent
    ```
    *ou*
    ```bash
    tail -f /var/log/glpi-agent/glpi-agent.log
    ```
* **Forcer manuellement un inventaire** (pour valider le flux réseau vers `deb-server-2`) :
    ```bash
    glpi-agent --server http://deb-server-2/front/inventory.php --force
    ```
* **Vérifier le flux réseau (API GitHub)** (en cas d'échec de téléchargement) :
    ```bash
    curl -I [https://api.github.com/repos/glpi-project/glpi-agent/releases/latest](https://api.github.com/repos/glpi-project/glpi-agent/releases/latest)
    ```
