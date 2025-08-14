---
title: "Installing a Nextcloud Server on Ubuntu"
description: "Complete guide to deploying a Nextcloud server with Apache, PHP, and MariaDB on Ubuntu."
date: 2025-08-12
draft: false
---

{{< imgx src="img/labs/nextcloud/nextcloud_logo.png" alt="Nextcloud" width="160px" align="center" >}}

## Objective
In this tutorial, we will see how to set up a functional local Nextcloud server on Ubuntu using Apache2, PHP, and MariaDB.

In another tutorial, I will explain how to set up a Twingate connector to securely connect to our private cloud over the internet.

---

## Tools Used
- A Linux Ubuntu server (VM on Proxmox 8)
- LAMP (Linux, Apache2, MySQL, PHP)
- Nextcloud Server

## Detailed Steps

### 1. Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Download the latest version of Nextcloud
```bash
wget https://download.nextcloud.com/server/releases/latest.zip
```

### 3. Install MariaDB and secure it
```bash
sudo apt install mariadb-server -y
sudo mysql_secure_installation
sudo mariadb
```

### 4. Inside MariaDB
```sql
CREATE DATABASE nextcloud;
GRANT ALL ON nextcloud.* TO 'nextcloudadmin'@'localhost' IDENTIFIED BY 'STRONG_PASSWD!';
FLUSH PRIVILEGES;
EXIT;
```

### 5. Install PHP and required modules
```bash
sudo apt install php php-apcu php-bcmath php-cli php-common php-curl php-gd php-gmp php-imagick php-intl php-mbstring php-mysql php-zip php-xml -y
```

### 6. Enable Apache2 modules
```bash
sudo a2enmod dir env headers mime rewrite ssl
sudo systemctl reload apache2
sudo phpenmod bcmath gmp imagick intl
```

### 7. Extract and configure Nextcloud
```bash
sudo apt install unzip
sudo unzip latest.zip
sudo chown www-data:www-data -R nextcloud/
sudo mv nextcloud /var/www
```

### 8. Apache2 Configuration
#### 1. Disable the default site:
```bash
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```
#### 2. Create `/etc/apache2/sites-available/nextcloud.conf`:
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
#### 3. Enable the site:
```bash
sudo a2ensite nextcloud.conf
```

### 9. PHP Configuration
In the file `/etc/php/8.1/apache2/php.ini`, apply the following changes (according to your needs):

#### Lines to modify:
- Line 409 → `max_execution_time = 1500`
- Line 430 → `memory_limit = 512M`
- Line 698 → `post_max_size` & `upload_max_filesize = 10000M`
- Line 850 → `upload_max_filesize = 10000M`

#### Lines to uncomment:
- Line 968 → `date.timezone = Europe/Paris`
- Line 1767 → `opcache.enable=1`
- Line 1773 → `opcache.memory_consumption=128M`
- Line 1776 → `opcache.interned_strings_buffer=8`
- Line 1780 → `opcache.max_accelerated_files=10000`
- Line 1798 → `opcache.revalidate_freq=2`
- Line 1805 → `opcache.save_comments=1`

### Restart Apache2
```bash
sudo systemctl restart apache2
```

### Done !
Once we've restarted the apache services, our Nextcloud server is ready to use.