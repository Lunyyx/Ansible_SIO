# Déploiement GLPI Agent avec Ansible

Ce projet permet de déployer automatiquement l'agent GLPI sur plusieurs serveurs Linux via Ansible.

---

# Architecture

Voici un exemple d'architecture :

* Serveur Ansible : `DEB-ANSIBLE-1`
* Serveur GLPI : `deb-server-2`
* Machines cibles :

  * deb-server-1
  * deb-server-2
  * deb-web-1
  * deb-web-2

---

# Configuration du serveur GLPI (deb-server-2)

Le serveur `deb-server-2` héberge GLPI et doit être accessible par les agents.

## Vérification de l'accès à GLPI

Depuis le serveur Ansible ou une machine cliente :

```bash
curl http://deb-server-2/front/inventory.php
```

Une réponse HTTP doit être retournée.

---

## Vérification du service web

Sur `deb-server-2` :

```bash
sudo systemctl status apache2
```

ou :

```bash
sudo systemctl status nginx
```

Le service doit être actif.

---

## Vérification du pare-feu

S'assurer que les ports HTTP/HTTPS sont accessibles :

```bash
sudo ufw allow 80
sudo ufw allow 443
```

Vérifier également les règles de filtrage réseau (pare-feu, VLAN, etc.).

---

## Vérification de l'URL GLPI

Dans le fichier :

```
group_vars/linux_clients.yml
```

```yaml
glpi_server: "http://deb-server-2/front/inventory.php"
glpi_verify_ssl: false
glpi_agent_version: "latest"
```

Adapter si nécessaire :

* utiliser une adresse IP
* utiliser HTTPS si configuré

---

## Test depuis une machine cible

```bash
curl http://deb-server-2/front/inventory.php
```

Si la requête échoue, vérifier :

* la résolution DNS
* la connectivité réseau
* l'état du service web

---

# Préparation de l'accès SSH

## Génération de la clé SSH

Sur le serveur Ansible :

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible -C "ansible@server"
```

---

## Création de l'utilisateur distant

Sur chaque machine cible :

```bash
sudo adduser ansible
sudo usermod -aG sudo ansible
```

---

## Configuration du sudo sans mot de passe

```bash
sudo visudo
```

Ajouter la ligne suivante :

```
ansible ALL=(ALL) NOPASSWD:ALL
```

---

## Déploiement de la clé publique

Depuis le serveur Ansible :

```bash
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-server-1
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-server-2
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-web-1
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-web-2
```

---

## Test de connexion SSH

```bash
ssh ansible@deb-server-1
```

La connexion doit s'effectuer sans demande de mot de passe.

---

# Configuration Ansible

## Inventory

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

---

## Configuration globale

Fichier : `ansible.cfg`

```ini
[defaults]
roles_path = ./roles
```

---

# Déploiement

## Test de connectivité

```bash
ansible linux_clients -i inventory/production.ini -m ping
```

---

## Exécution du playbook

```bash
ansible-playbook -i inventory/production.ini playbooks/deploy-glpi-agent.yml
```

---

# Fonctionnement du rôle

Le rôle `glpi-agent` effectue les actions suivantes :

1. Installation des dépendances (`curl`, `perl`)
2. Récupération de la dernière version du GLPI Agent
3. Téléchargement de l'installateur
4. Installation de l'agent
5. Déploiement du fichier de configuration
6. Activation et démarrage du service
7. Envoi d'un inventaire initial

---

# Configuration GLPI Agent

Template : `roles/glpi-agent/templates/agent.cfg.j2`

Paramètres principaux :

* `server` : URL du serveur GLPI
* `no-ssl-check` : désactivation de la vérification SSL
* `tag` : identification de l'agent
* `delaytime` : fréquence d'exécution

---

# Dépannage

## Problème SSH

```bash
ssh -i ~/.ssh/ansible ansible@deb-server-1
```

---

## Problème Ansible

```bash
ansible linux_clients -m ping --become -vvv
```

---

## Service GLPI Agent

```bash
systemctl status glpi-agent
```

---

## Téléchargement du package

```bash
curl https://api.github.com/repos/glpi-project/glpi-agent/releases/latest
```

---

# Sécurisation (optionnel)

Sur les machines cibles :

```bash
sudo nano /etc/ssh/sshd_config
```

Modifier les paramètres suivants :

```
PasswordAuthentication no
PermitRootLogin no
```

Redémarrer le service SSH :

```bash
sudo systemctl restart ssh
```

---

# Structure du projet

```
/opt/ansible
├── ansible.cfg
├── group_vars/
├── inventory/
├── playbooks/
└── roles/
```

---

# Résumé

* Accès SSH sécurisé par clé
* Utilisateur dédié `ansible`
* Déploiement automatisé du GLPI Agent
* Serveur GLPI centralisé (`deb-server-2`)
* Configuration centralisée via Ansible

---

# Objectif

Automatiser l'installation et la configuration du GLPI Agent sur un parc de serveurs Linux de manière fiable et reproductible.
