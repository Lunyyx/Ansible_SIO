# Déploiement GLPI Agent avec Ansible

Ce projet permet de déployer automatiquement l'agent GLPI sur plusieurs serveurs Linux via Ansible.

---

# Architecture

Voici un exemple d'architecture:

* Serveur Ansible : `DEB-ANSIBLE-1`
* Serveur GLPI : `deb-server-2`
* Machines cibles :

  * deb-server-1
  * deb-server-2
  * deb-web-1
  * deb-web-2

---

# Préparation de l'accès SSH

## 1. Générer une clé SSH dédiée

Sur le serveur Ansible :

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible -C "ansible@server"
```

---

## 2. Créer un utilisateur sur chaque machine cible

```bash
sudo adduser ansible
sudo usermod -aG sudo ansible
```

---

## 3. Autoriser le sudo sans mot de passe

```bash
sudo visudo
```

Ajouter :

```
ansible ALL=(ALL) NOPASSWD:ALL
```

---

## 4. Copier la clé publique

Depuis le serveur Ansible :

```bash
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-server-1
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-server-2
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-web-1
ssh-copy-id -i ~/.ssh/ansible.pub ansible@deb-web-2
```

---

## 5. Tester la connexion SSH

```bash
ssh ansible@deb-server-1
```

La connexion doit se faire sans mot de passe

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

## Variables globales

Fichier : `group_vars/linux_clients.yml`

```yaml
glpi_server: "http://deb-server-2/front/inventory.php"
glpi_verify_ssl: false
glpi_agent_version: "latest"
```

---

## Configuration Ansible

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

## Lancer le déploiement

```bash
ansible-playbook -i inventory/production.ini playbooks/deploy-glpi-agent.yml
```

---

# Fonctionnement du rôle

Le rôle `glpi-agent` :

1. Installe les dépendances (`curl`, `perl`)
2. Récupère automatiquement la dernière version du GLPI Agent
3. Télécharge l'installer
4. Installe l'agent
5. Déploie la configuration (`/etc/glpi-agent/agent.cfg`)
6. Active et démarre le service
7. Force un inventaire immédiat

---

# Configuration GLPI Agent

Template : `roles/glpi-agent/templates/agent.cfg.j2`

Options utilisées :

* `server` : URL du serveur GLPI
* `no-ssl-check` : désactivation SSL si nécessaire
* `tag` : tag d’identification
* `delaytime` : fréquence d’inventaire

---

# Dépannage

## SSH ne fonctionne pas

```bash
ssh -i ~/.ssh/ansible ansible@deb-server-1
```

---

## Ansible ne passe pas

```bash
ansible linux_clients -m ping --become -vvv
```

---

## GLPI agent non actif

```bash
systemctl status glpi-agent
```

---

## Problème téléchargement

Tester manuellement :

```bash
curl https://api.github.com/repos/glpi-project/glpi-agent/releases/latest
```

---

# Sécurisation (optionnel)

Sur les machines cibles :

```bash
sudo nano /etc/ssh/sshd_config
```

Modifier :

```
PasswordAuthentication no
PermitRootLogin no
```

Puis :

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
* Déploiement automatisé GLPI Agent
* Configuration centralisée via Ansible

---

# Objectif

Automatiser l'installation et la configuration du GLPI Agent sur un parc de serveurs Linux de manière fiable et reproductible.
