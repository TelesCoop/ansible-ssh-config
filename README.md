# Configuration Ansible pour serveurs TelesCoop

Ce dépôt contient des playbooks Ansible pour configurer et maintenir l'infrastructure des serveurs TelesCoop.
Plus globalement, il automatise la configuration sécurisée des serveurs Ubuntu 22+ avec gestion des utilisateurs, SSH, sécurité et services essentiels.


## Objectif

- Configuration sécurisée des serveurs (SSH, utilisateurs, permissions)
- Gestion automatisée des employés via GitHub
- Installation et configuration des services essentiels
- Configuration des notifications par email via Mailgun
- Mise à jour automatique des systèmes
- Génération de clés SSH pour GitHub CI/CD

## Prérequis

- Ansible installé sur votre machine locale
- Accès SSH root aux serveurs cibles (temporaire, sera désactivé après configuration)
- Fichier `vault.key` pour les mots de passe chiffrés (optionnel pour mail)
- Collection `community.general` pour les modules snap

## Structure du projet

```
.
├── server-config.yml          # Playbook principal de configuration
├── mail.yml                   # Configuration mail Mailgun
├── hosts                      # Inventaire des serveurs
├── group_vars/all/vars.yaml   # Variables globales
├── templates/                 # Templates de configuration
├── roles/                     # Rôles Ansible
└── ansible.cfg               # Configuration Ansible
```

## Configurer un nouveau serveur

### 1. Préparation du serveur

Avant d'utiliser ce template, assurez-vous que votre nouveau serveur :

```bash
# Sur le serveur cible
# 1. Ubuntu 22.04+ est installé
# 2. Accès SSH root temporaire est configuré
# 3. Le serveur a accès à internet pour télécharger les packages et le fichier employees.yaml

# Note : Les utilisateurs seront automatiquement créés depuis le fichier
# employees.yaml du dépôt GitHub TelesCoop/company-settings
# Pas besoin de créer manuellement des utilisateurs
```

### 2. Ajout à l'inventaire

Ajoutez votre nouveau serveur dans le fichier `hosts` :

```ini
[nouveau_serveur]
nom-serveur.domaine.com:42722 ansible_user=ubuntu

# ou pour un serveur avec port SSH standard
[nouveau_serveur]
nom-serveur.domaine.com ansible_user=ubuntu
```

### 3. Configuration des variables

Vérifiez les variables dans `group_vars/all/vars.yaml` :

```yaml
# Port SSH personnalisé (adapté selon votre config)
sshd:
  port: 42722

# Adresse email pour les notifications
support_mail_addr: votre-email@domaine.com

# Chemin de la clé SSH pour GitHub CI/CD
github_ssh_key: /home/{{ ansible_user }}/.ssh/id_ed25519_github
```

**Note** : Certains serveurs comme vps03 ont des exceptions (ex: pas d'installation de packages ou configuration SSH différente).

### 4. Test de connectivité

```bash
# Tester la connexion au nouveau serveur
ansible nouveau_serveur -m ping

# Tester tous les serveurs
ansible all -m ping
```

### 5. Déploiement

```bash
# Configuration complète du nouveau serveur
ansible-playbook server-config.yml --limit nouveau_serveur

# Optionnel : Configuration mail
ansible-playbook mail.yml --limit nouveau_serveur
```

## Utilisation sur serveurs existants

### 1. Configuration initiale des serveurs

```bash
# Vérifier l'accès aux serveurs
ansible all -m ping

# Configurer SSH et installer les packages essentiels
ansible-playbook server-config.yml
```

### 2. Configuration du mail (optionnel)

```bash
# Configurer Mailgun pour les notifications
ansible-playbook mail.yml
```

### 3. Configuration sans mail

Si vous n'avez pas besoin de la configuration mail, commentez cette ligne dans `ansible.cfg` :

```ini
# vault_password_file = vault.key
```

## Fonctionnalités

### Sécurité
- Désactivation de l'accès SSH root
- Configuration SSH sécurisée via template
- Installation et configuration de fail2ban
- Gestion des groupes d'utilisateurs (devops, ssh-allowed)
- Configuration des permissions sudo

### Gestion des utilisateurs
- Téléchargement automatique du fichier `employees.yaml` depuis le dépôt GitHub `TelesCoop/company-settings`
- Création automatique des utilisateurs listés dans ce fichier
- Configuration des clés SSH pour chaque utilisateur (sur leur compte et sur ansible_user)
- Gestion des permissions sudo (groupe devops, ajout au groupe sudo)
- Ajout au groupe ssh-allowed pour restreindre l'accès SSH
- Suppression automatique des anciens utilisateurs (marqués `remove: true`)

### Services essentiels
- Installation des packages de base (iptables, fail2ban, nginx, certbot, etc.)
- Configuration nginx avec template par défaut
- Gestion des certificats SSL (certbot avec renouvellement automatique)
- Configuration des mises à jour automatiques (unattended-upgrades)
- Synchronisation temporelle (systemd-timesyncd, timezone Europe/Paris)
- Configuration du hostname basé sur le nom du groupe

### Monitoring
- Configuration des notifications mail via mdadm
- Surveillance RAID (mdadm avec alertes email)
- Logrotate pour nginx
- Redémarrage automatique le lundi matin à 6h si nécessaire
- Génération de clé SSH pour GitHub CI/CD

## Serveurs gérés

Le fichier `hosts` contient l'inventaire des serveurs TelesCoop :
- vps00 à vps06 : Serveurs principaux
- s00 : Serveur spécialisé

## Variables importantes

- `support_mail_addr` : Adresse email pour les notifications
- `github_ssh_key` : Clé SSH pour GitHub CI/CD
- `mailgun_smtp_password` : Mot de passe SMTP chiffré (dans vault)

## Ajouter/Supprimer des utilisateurs

Pour ajouter ou supprimer des utilisateurs :

1. **Modifier le fichier `employees.yaml`** dans le dépôt GitHub `TelesCoop/company-settings`
2. **Ajouter un utilisateur** :
   ```yaml
   employees:
     - name: nouvel_utilisateur
       ssh_key: "ssh-rsa AAAAB3NzaC1yc2E..."
   ```
3. **Supprimer un utilisateur** :
   ```yaml
   employees:
     - name: ancien_utilisateur
       remove: true
   ```
4. **Relancer le playbook** : `ansible-playbook server-config.yml`

Les utilisateurs seront automatiquement créés/supprimés sur tous les serveurs lors du prochain déploiement.

### Changer l'URL du dépôt des employés

Si vous devez utiliser un autre dépôt pour le fichier `employees.yaml`, modifiez directement l'URL dans le fichier `server-config.yml` :

```yaml
- name: Download employees file
  get_url:
    url: https://raw.githubusercontent.com/VOTRE_ORG/VOTRE_REPO/main/employees.yaml
    dest: ./employees.yaml
    force: true
```

Remplacez `VOTRE_ORG/VOTRE_REPO` par l'organisation et le nom de votre dépôt.

## Maintenance

Les serveurs sont configurés pour :
- Redémarrer automatiquement le lundi matin si des mises à jour le nécessitent
- Renouveler les certificats SSL hebdomadairement
- Effectuer les mises à jour de sécurité automatiquement
