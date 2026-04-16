# Déploiement Automatisé de l'Agent GLPI avec Ansible

## Présentation
Ce projet Ansible permet d'automatiser l'installation et la configuration de l'agent GLPI sur des parcs Linux (Debian/Ubuntu). Il gère l'intégralité de la chaîne de déploiement : de la création des accès SSH sécurisés à la remontée de l'inventaire final.

## Architecture du Projet
L'arborescence est organisée de manière modulaire :

- **group_vars/linux_clients.yml** : Configuration globale du serveur GLPI.
- **inventory/production.ini** : Liste des adresses IP/Hostnames et paramètres de connexion.
- **roles/glpi-agent/** : Logique d'installation (dépendances, téléchargement, configuration).
- **playbooks/deploy-glpi-agent.yml** : Orchestration des rôles.

## Initialisation de l'Infrastructure (Bootstrap)

Avant de déployer l'agent, les serveurs cibles doivent être accessibles. Le projet gère les deux étapes critiques suivantes :

### 1. Création de l'utilisateur et accès Sudo
Le playbook s'assure de la présence d'un utilisateur dédié (ex: `ansible`) sur les machines de destination. Cet utilisateur est ajouté au groupe `sudo` pour permettre l'installation des paquets système.

### 2. Génération et Déploiement des Clés SSH
Pour permettre une automatisation sans mot de passe, la procédure suivante est suivie :
- **Génération** : Une paire de clés SSH (RSA 4096 bits) est générée sur le nœud de contrôle Ansible.
- **Distribution** : La clé publique est injectée dans le fichier `/home/ansible/.ssh/authorized_keys` de chaque client.
- **Vérification** : Le projet valide que la connexion SSH est opérationnelle avant de lancer l'installation de l'agent.

## Configuration du Serveur GLPI

L'adresse du serveur GLPI est définie à deux endroits pour garantir la cohérence du déploiement. Vous devez modifier la variable `glpi_server` dans :

1.  **Priorité haute** : `/opt/ansible/group_vars/linux_clients.yml`
2.  **Valeur par défaut** : `/opt/ansible/roles/glpi-agent/defaults/main.yml`

*Exemple de valeur : `http://votre-serveur-glpi/front/inventory.php`*

## Variables Principales

| Variable | Description |
| :--- | :--- |
| `glpi_server` | Point de terminaison du serveur GLPI. |
| `glpi_verify_ssl` | `false` pour autoriser les serveurs sans certificat valide. |
| `ansible_user` | Utilisateur distant créé pour le déploiement. |
| `ansible_ssh_private_key_file` | Chemin vers la clé privée générée. |

## Instructions d'Utilisation

### Étape 1 : Préparation des clés
Si vous n'avez pas encore de clé, générez-la sur votre machine de contrôle :
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible
```

### Étape 2 : Injection initiale
Pour la première connexion (si les clés ne sont pas encore déployées), utilisez la commande suivante pour copier la clé sur vos clients :
```bash
ssh-copy-id -i ~/.ssh/ansible.pub ansible@adresse-du-client
```

### Étape 3 : Lancement du déploiement
Lancez le playbook pour créer les utilisateurs, configurer le SSH et installer l'agent GLPI :
```bash
ansible-playbook -i inventory/production.ini playbooks/deploy-glpi-agent.yml
```

## Maintenance et Diagnostics
- **Vérification du service** : `systemctl status glpi-agent`
- **Déclenchement manuel** : `glpi-agent --force`
- **Logs** : `/var/log/glpi-agent/glpi-agent.log`
