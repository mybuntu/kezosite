---
title: "Installation d’un serveur Nextcloud sur Ubuntu"
description: "Guide complet pour déployer un serveur Nextcloud avec Apache, PHP et MariaDB sur Ubuntu."
date: 2025-08-12
draft: false
---
{{< imgx src="img/labs/nextcloud/nextcloud_logo.png" alt="Nextcloud" width="160px" align="center" >}}
## Objectif
Dans ce tuto nous verrons comment mettre en place un serveur Nextcloud local fonctionnel sur Ubuntu avec Apache2, PHP et MariaDB. 

Dans un autre tuto j'expliquerai comment mettre en place un connecteur Twingate afin de joindre notre cloud privé de manière sécurisée par internet.

---
## Outils utilisé
- Un serveur Linux Ubuntu (VM sur proxmox 8)
- LAMP (Linux, Apache2, Mysql, PHP)
- Nextcloud Server 

## Étapes détaillées

### 1. Mise à jour du système 
```bash
sudo apt update && sudo apt upgrade -y
```
### 2. Téléchargement de la dernière version de Nextcloud
```bash
wget https://download.nextcloud.com/server/releases/latest.zip
```
## 3. Installation de MariaDB et sécurisation
```bash
sudo apt install mariadb-server -y
sudo mysql_secure_installation
sudo mariadb
```
### 4. Dans MariaDB 
```sql
CREATE DATABASE nextcloud;
GRANT ALL ON nextcloud.* TO 'nextcloudadmin'@'localhost' IDENTIFIED BY 'STRONG_PASSWD!';
FLUSH PRIVILEGES;
EXIT;
```
### 5. Installation de PHP et modules nécessaires
```bash 
sudo apt install php php-apcu php-bcmath php-cli php-common php-curl php-gd php-gmp php-imagick php-intl php-mbstring php-mysql php-zip php-xml -y
```
### 6. Activation de modules Apache2
```bash 
sudo a2enmod dir env headers mime rewrite ssl
sudo systemctl reload apache2
sudo phpenmod bcmath gmp imagick intl
```
### 7. Décompression et configuration Nextcloud
```bash
sudo apt install unzip
sudo unzip latest.zip
sudo chown www-data:www-data -R nextcloud/
sudo mv nextcloud /var/www
```
### 8. Configuration Apache2
#### 8.1. Désactivation du site par défaut :
```bash
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```
#### 8.2. Création du fichier /etc/apache2/sites-available/nextcloud.conf :
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/nextcloud/
    <Directory "/var/www/nextcloud/">
        Options MultiViews FollowSymlinks
        AllowOverride All
        Order allow,deny
        Allow from all
    </Directory>
    TransferLog /var/log/apache2/nextcloud_access.log
    ErrorLog /var/log/apache2/nextcloud_error.log
</VirtualHost>
```
#### 8.3. Activation du site :
```bash 
sudo a2ensite nextcloud.conf
```
### 9. Configuration PHP
Dans le fichier : `/etc/php/8.1/apache2/php.ini`, appliquer les modifications suivantes (Selon les besoins) :

Dans ces moments là, il est toujours préférable pour économiser du temps et s'éviter un processus de réinstallation totale, de garder une version de secours du fichier `.ini` par défaut. Pour cela, on execute la commande suivante : 
```bash
sudo cp /etc/php/8.1/apache2/php.ini /etc/php/8.1/apache2/php.ini.backedup
```

#### Lignes à Modifier : 
- Line 409 → `max_execution_time = 1500`
- Line 430 → `memory_limit = 512M`
- Line 698 → `post_max_size` & `upload_max_filesize = 10000M`
- Line 850 → `upload_max_filesize = 10000M`
#### Lignes à décommenter : 
- Line 968 → `date.timezone = Europe/Paris`
- Line 1767 → `opcache.enable=1`
- Line 1773 → `opcache.memory_consumption=128M`
- Line 1776 → `opcache.interned_strings_buffer=8`
- Line 1780 → `opcache.max_accelerated_files=10000`
- Line 1798 → `opcache.revalidate_freq=2`
- Line 1805 → `opcache.save_comments=1`

### Redémarrage d'Apache2
```bash
sudo systemctl restart apache2
```
### C'est fini
Après le redémarrage des services Apache le serveur est en place et prêt à l'emploi. 