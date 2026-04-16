# Déploiement Automatisé de l'Agent GLPI avec Ansible

## Présentation
Ce projet Ansible automatise l'installation, la configuration et le déploiement de l'agent GLPI sur des clients Linux (distributions basées sur Debian/Ubuntu). Il intègre la gestion complète du cycle de vie de l'agent, depuis la préparation des accès système jusqu'à l'envoi du premier inventaire.

## Architecture du Projet
Le projet est structuré selon les standards Ansible pour garantir une maintenance simplifiée :

- **group_vars/linux_clients.yml** : Variables globales pour le groupe de clients.
- **inventory/production.ini** : Définition des hôtes cibles et des paramètres de connexion.
- **roles/glpi-agent/** : Rôle contenant la logique d'installation, les templates de configuration et les gestionnaires de services.
- **playbooks/deploy-glpi-agent.yml** : Playbook principal orchestrant le déploiement.

## Gestion des Accès et Sécurité

### Création des Utilisateurs
Le projet automatise la création des comptes utilisateurs sur les serveurs de destination. Il s'assure que l'utilisateur configuré (par défaut `ansible`) possède les droits nécessaires pour l'administration du système via sudo.

### Configuration SSH
La sécurité des communications est assurée par :
1. La génération d'une paire de clés SSH (si inexistante).
2. Le déploiement de la clé publique sur les serveurs cibles dans le fichier `authorized_keys`.
3. L'utilisation de ces clés pour les connexions automatisées sans mot de passe.

## Configuration du Serveur GLPI

Conformément aux exigences du projet, l'adresse du serveur GLPI doit être configurée de manière cohérente à deux emplacements distincts pour assurer la redondance et la priorité des variables :

1. **Priorité haute** : Dans `/opt/ansible/group_vars/linux_clients.yml`.
2. **Valeur par défaut** : Dans `/opt/ansible/roles/glpi-agent/defaults/main.yml`.

Assurez-vous que la variable `glpi_server` pointe vers votre point de terminaison d'inventaire (ex: `http://votre-serveur/front/inventory.php`).

## Variables Principales

| Variable | Description |
| :--- | :--- |
| `glpi_server` | URL de destination pour les inventaires GLPI. |
| `glpi_verify_ssl` | Désactivation de la vérification SSL (utile pour les certificats auto-signés). |
| `glpi_agent_version` | Version de l'agent à installer (par défaut `latest`). |
| `ansible_user` | Utilisateur système utilisé pour le déploiement. |
| `ansible_ssh_private_key_file` | Chemin vers la clé privée SSH sur le nœud de contrôle. |

## Instructions d'Utilisation

### Prérequis
- Ansible installé sur la machine de contrôle.
- Accès réseau vers les clients sur le port 22 (SSH).

### Exécution
Pour lancer le déploiement complet, exécutez la commande suivante depuis la racine du projet :

```bash
ansible-playbook -i inventory/production.ini playbooks/deploy-glpi-agent.yml
```

### Étapes automatisées
1. **Provisionnement** : Création de l'utilisateur et déploiement des clés SSH.
2. **Installation des dépendances** : Installation de `curl` et `perl`.
3. **Récupération** : Identification et téléchargement de la dernière version de l'agent depuis GitHub.
4. **Configuration** : Génération du fichier `/etc/glpi-agent/agent.cfg` via template J2.
5. **Initialisation** : Activation du service système et déclenchement d'un inventaire immédiat.

## Maintenance
- **Statut du service** : `systemctl status glpi-agent`
- **Logs de l'agent** : `/var/log/glpi-agent/glpi-agent.log`
- **Forcer un inventaire** : `glpi-agent --server http://votre-serveur/front/inventory.php`
