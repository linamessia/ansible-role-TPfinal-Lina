# ansible-role-wordpress

Rôle Ansible pour déployer automatiquement **WordPress** avec **MariaDB** et **Apache/httpd** sur Ubuntu et Rocky Linux, y compris dans des environnements **conteneurisés** 

Ce rôle est la traduction en Ansible idempotent du script `install_wordpress.sh` fourni en contexte professionnel DevOps.

---

## Fonctionnalités

- ✅ Compatible **Ubuntu 20.04 / 22.04** et **Rocky Linux 8 / 9**
- ✅ **Idempotent** : peut être relancé plusieurs fois sans effet de bord
- ✅ Détection automatique de l'OS (`ansible_os_family`)
- ✅ Installation et sécurisation de **MariaDB** (suppression users anonymes, base test)
- ✅ Téléchargement et extraction de **WordPress** (version configurable)
- ✅ Génération de `wp-config.php` avec **salts uniques** via l'API WordPress
- ✅ Configuration du **VirtualHost Apache/httpd** via template Jinja2
- ✅ Installation de **WP-CLI** et configuration initiale via CLI
- ✅ **Handlers** : reload/restart Apache et MariaDB déclenchés uniquement si nécessaire
- ✅ Variables entièrement configurables via `defaults/`

---

## Prérequis

### Ansible

```bash
ansible --version   # >= 2.12
```

### Collections Galaxy requises

```bash
ansible-galaxy collection install community.mysql ansible.posix
```

### Accès cible

- Connexion SSH (ou Docker) avec élévation `become: true`
- Python 3 disponible sur les cibles

---

## Structure du rôle

```
ansible-role-wordpress/
├── defaults/
│   └── main.yml              # Variables par défaut (toutes surchargeables)
├── vars/
│   ├── main.yml              # Variables internes
│   ├── Debian.yml            # Paquets et chemins Ubuntu/Debian
│   └── RedHat.yml            # Paquets et chemins Rocky Linux/RHEL
├── tasks/
│   ├── main.yml              # Orchestrateur — inclut les sous-tâches
│   ├── packages.yml          # Installation des paquets système
│   ├── mariadb.yml           # Configuration et sécurisation MariaDB
│   ├── wordpress_download.yml# Téléchargement et extraction WordPress
│   ├── wordpress_config.yml  # Génération wp-config.php
│   ├── apache.yml            # VirtualHost + activation modules
│   ├── services.yml          # Démarrage des services
│   └── wpcli.yml             # WP-CLI + installation WordPress
├── handlers/
│   └── main.yml              # reload/restart Apache et MariaDB
├── templates/
│   └── wordpress.conf.j2     # VirtualHost Apache/httpd (Jinja2)
├── meta/
│   └── main.yml              # Métadonnées Ansible Galaxy
├── playbook.yml              # Playbook d'exemple
└── inventory.ini             # Inventaire d'exemple
```

---

## Variables

### Variables par défaut (`defaults/main.yml`)

Toutes ces variables peuvent être surchargées dans `host_vars`, `group_vars` ou directement dans le playbook.

| Variable | Défaut | Description |
|---|---|---|
| `wordpress_version` | `latest` | Version WordPress (`latest` ou ex: `6.5.3`) |
| `wordpress_install_dir` | `/var/www/html` | Répertoire DocumentRoot |
| `wordpress_url` | `http://localhost` | URL publique du site |
| `wordpress_db_name` | `wordpress` | Nom de la base de données |
| `wordpress_db_user` | `wp_user` | Utilisateur MariaDB WordPress |
| `wordpress_db_password` | `ChangeMe_S3cur3!` | **À changer !** Mot de passe DB |
| `wordpress_db_host` | `localhost` | Hôte MariaDB |
| `mariadb_root_password` | `RootPass_S3cur3!` | **À changer !** Mot de passe root MariaDB |
| `wordpress_admin_user` | `admin` | Login administrateur WordPress |
| `wordpress_admin_password` | `AdminPass_S3cur3!` | **À changer !** Mot de passe admin |
| `wordpress_admin_email` | `admin@example.com` | Email administrateur |
| `wordpress_site_title` | `Mon Site WordPress` | Titre du site |
| `wordpress_locale` | `fr_FR` | Langue (`fr_FR`, `en_US`, etc.) |

> ⚠️ **Sécurité** : ne jamais laisser les mots de passe par défaut en production. Utilisez `ansible-vault` pour chiffrer les secrets.

### Variables OS-spécifiques (`vars/Debian.yml` et `vars/RedHat.yml`)

Ces variables sont chargées automatiquement selon l'OS détecté et ne sont pas destinées à être surchargées.

| Variable | Ubuntu | Rocky Linux |
|---|---|---|
| `apache_service` | `apache2` | `httpd` |
| `apache_conf_dir` | `/etc/apache2/sites-available` | `/etc/httpd/conf.d` |
| `apache_user` | `www-data` | `apache` |
| `apache_group` | `www-data` | `apache` |
| `wordpress_packages` | `apache2, libapache2-mod-php, php-mysql...` | `httpd, php-mysqlnd...` |

---

## Correspondance avec le script original

| Étape du script `install_wordpress.sh` | Fichier Ansible correspondant |
|---|---|
| `apt update && apt install apache2 php ...` | `tasks/packages.yml` |
| `rm -f /var/www/html/index.html` | `tasks/wordpress_download.yml` |
| `service apache2 restart` / `mysqld_safe &` | `tasks/services.yml` + handlers |
| `mysql -e "ALTER USER 'root'..."` | `tasks/mariadb.yml` |
| `mysql -e "CREATE DATABASE wordpress"` | `tasks/mariadb.yml` |
| `wget latest.zip && unzip && cp -r wordpress/*` | `tasks/wordpress_download.yml` |
| `chown -R www-data && chmod -R 755` | `tasks/wordpress_download.yml` |
| `cp wp-config-sample.php wp-config.php` | `tasks/wordpress_config.yml` |
| `sed -i "s/database_name_here/..."` | `tasks/wordpress_config.yml` |
| `cat > wordpress.conf << EOF ... EOF` | `templates/wordpress.conf.j2` |
| `a2ensite wordpress.conf` | `tasks/apache.yml` |
| `a2enmod rewrite` | `tasks/apache.yml` + `packages.yml` |
| `service apache2 reload` | Handler `reload apache` |

---

## Utilisation

### 1. Installer le rôle depuis Galaxy

```bash
ansible-galaxy role install votre_namespace.wordpress
```

Ou via un fichier `requirements.yml` :

```yaml
roles:
  - name: votre_namespace.wordpress
    version: "1.0.0"
```

```bash
ansible-galaxy install -r requirements.yml
```

### 2. Inventaire (`inventory.ini`)

```ini
[wordpress_servers]
ubuntu-wp   ansible_host=172.17.0.2  ansible_user=root  ansible_connection=docker
rocky-wp    ansible_host=172.17.0.3  ansible_user=root  ansible_connection=docker

[wordpress_servers:vars]
ansible_python_interpreter=/usr/bin/python3
```

### 3. Playbook (`playbook.yml`)

```yaml
- name: "Déploiement WordPress"
  hosts: wordpress_servers
  become: true
  gather_facts: true

  vars:
    wordpress_db_password: "MonMotDePasse_2024!"
    mariadb_root_password: "MonRootMDP_2024!"
    wordpress_admin_email: "admin@mondomaine.fr"
    wordpress_site_title: "Mon Blog"
    wordpress_url: "http://{{ ansible_default_ipv4.address }}"

  roles:
    - role: votre_namespace.wordpress
```

### 4. Lancer le déploiement

```bash
# Déploiement complet
ansible-playbook playbook.yml -i inventory.ini

# Avec vérification préalable (dry-run)
ansible-playbook playbook.yml -i inventory.ini --check

# Déploiement partiel par tag
ansible-playbook playbook.yml -i inventory.ini --tags mariadb
ansible-playbook playbook.yml -i inventory.ini --tags wordpress
ansible-playbook playbook.yml -i inventory.ini --tags apache

# Surcharger une variable à la volée
ansible-playbook playbook.yml -i inventory.ini -e "wordpress_site_title='Mon nouveau titre'"
```

---

## Tags disponibles

| Tag | Tâches exécutées |
|---|---|
| `packages` | Installation des paquets système |
| `mariadb` | Configuration MariaDB complète |
| `wordpress` | Téléchargement + wp-config.php |
| `apache` | VirtualHost + modules |
| `services` | Démarrage des services |
| `wpcli` | WP-CLI + installation WordPress |

---

## Notes pour les conteneurs

Les conteneurs n'ont généralement pas de `systemd`. Voici comment démarrer les services manuellement :

**MariaDB :**
```bash
mysqld_safe --datadir=/var/lib/mysql &
```

**Apache2 (Ubuntu) :**
```bash
service apache2 start
```

**httpd (Rocky Linux) :**
```bash
/usr/sbin/httpd -DFOREGROUND &
```

Le rôle utilise le module `ansible.builtin.service` qui s'adapte automatiquement à l'init system disponible. En cas d'échec sur un conteneur, assurez-vous que `dbus` et le service `init` sont disponibles ou démarrez les services manuellement avant d'exécuter le rôle.

---

## Sécurisation des secrets avec Ansible Vault

```bash
# Chiffrer les variables sensibles
ansible-vault encrypt_string 'MonMotDePasseSecurise!' --name 'wordpress_db_password'

# Lancer le playbook en demandant le mot de passe vault
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

---

## Publication sur Ansible Galaxy

### Prérequis

1. Compte sur [galaxy.ansible.com](https://galaxy.ansible.com)
2. Token API Galaxy (dans votre profil Galaxy)

### Étapes

```bash
# 1. Initialiser un namespace (fait via l'interface web Galaxy)

# 2. Configurer le token
ansible-galaxy setup --token VOTRE_TOKEN_GALAXY

# 3. Vérifier la structure du rôle
ansible-galaxy role init --offline votre_namespace.wordpress

# 4. Importer le rôle depuis GitHub
#    → Aller sur galaxy.ansible.com > My Content > Add Role
#    → Lier votre dépôt GitHub

# 5. Ou via CLI (après avoir poussé sur GitHub)
ansible-galaxy role import votre_namespace votre_repo_github
```

Le fichier `meta/main.yml` doit être correctement renseigné (auteur, plateformes, tags) avant l'import.


