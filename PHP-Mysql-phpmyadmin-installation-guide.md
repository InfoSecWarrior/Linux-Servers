# PHP, MySQL, and phpMyAdmin Installation Guide (CentOS/RHEL)

This document provides a complete and structured guide to install **PHP**, **MySQL**, and **phpMyAdmin** on CentOS/RHEL systems. The process includes repository setup, service management, and basic configuration.

---

## 🧠 Introduction

- **PHP (Hypertext Preprocessor):** A server-side scripting language widely used for web development.
- **MySQL:** An open-source relational database management system (RDBMS).
- **phpMyAdmin:** A web-based interface for managing MySQL databases using a browser.

---
## PHP Installation- 

## 🔹 Step 1: Install Required Repositories

### 🔸 Enable EPEL repository

Used to access additional open-source packages.

```bash
yum install epel-release -y
```

### 🔸 Install yum utilities

Provides additional package management features.

```bash
yum install yum-utils -y
```

### 🔸 Add REMI repository

Required for installing modern versions of PHP.

```bash
yum install -y http://rpms.remirepo.net/enterprise/remi-release-9.rpm
```

### 🔸 Enable REMI PHP module

Used to install a specific PHP version.

```bash
yum module enable php:remi-8.2 -y
```

### 🔸 Verify REMI installation

Confirms if the REMI repository was added.

```bash
rpm -qa | grep remi
```

---

## 🔹 Step 2: Install PHP and Required Extensions

### 🔸 Install default PHP version

Installs the default PHP version with all common modules.

```bash
yum install php*
```

### 🔸 Install specific PHP modules

Better control over which PHP modules get installed.

```bash
yum install -y \
php php-common php-cli php-fpm php-mysqlnd php-gd \
php-mbstring php-xml php-pdo php-opcache php-pear \
php-process php-pecl-http nginx-filesystem
```

### 🔸 Check PHP version

Ensures PHP is installed correctly.

```bash
php -v
```

### ✅ PHP installation is complete. Now we will configure it.

### 🔸 Restart Apache service

Applies PHP settings with web server.

```bash
systemctl restart httpd.service
```

### 🔸 Create PHP info file

Verifies PHP installation through browser.

```bash
vim /var/www/html/phpinfo.php
```

Paste this content:

```php
<?php
    phpinfo();
?>
```

### 🔸 View PHP info in browser

```
http://<server-ip>/phpinfo.php
```

---

## 🔹  MySQL Installation

### 🔸 Add MySQL repository

Needed to install the MySQL server.

```bash
yum install -y https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
```

### 🔸 Verify MySQL repo installation

```bash
rpm -qa | grep mysql
```

### 🔸 Install MySQL server

Installs the main MySQL service.

```bash
yum install -y mysql-community-server
```

### 🔸 Enable MySQL service

Ensures MySQL starts automatically on boot.

```bash
systemctl enable mysqld.service
```

### 🔸 Start MySQL service

Begins running the database server.

```bash
systemctl start mysqld.service
```

### 🔸 Check MySQL status

```bash
systemctl status mysqld.service
```

### 🔸 Get temporary root password

Used for first login to MySQL.

```bash
cat /var/log/mysqld.log | grep 'temporary password'
```

### 🔸 Secure MySQL installation

Sets root password and basic security settings.

```bash
mysql_secure_installation
```

### 🔸 Login to MySQL

```bash
mysql -u root -p
```

### ① (Optional) Change root password

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'anujexample';
```

### ✅ MySQL installation is complete. Now we will configure it.

---

## 🔹  phpMyAdmin Installation

### 🔸 Go to temp directory

```bash
cd /tmp
```

### 🔸 Download phpMyAdmin zip

```bash
wget https://files.phpmyadmin.net/phpMyAdmin/5.2.2/phpMyAdmin-5.2.2-all-languages.zip
```

### 🔸 Extract phpMyAdmin

```bash
unzip phpMyAdmin-5.2.2-all-languages.zip
```

### ✅ phpMyAdmin installation is complete. Now we will configure it.

### 🔸 Create target directory

```bash
mkdir  /var/www/html/phpmyadmin
```

### 🔸 Move files to web directory

```bash
mv phpMyAdmin-5.2.2-all-languages/* /var/www/html/phpmyadmin/
```

### 🔸 Copy sample config

```bash
cp /var/www/html/phpmyadmin/config.sample.inc.php /var/www/html/phpmyadmin/config.inc.php
```

### 🔸 Edit phpMyAdmin config file and set blowfish secret

Used to configure internal encryption for cookies.

```bash
vim /var/www/html/phpmyadmin/config.inc.php
```

### 🔸 Generate a 32-character random string (Blowfish secret)

```bash
pwgen 32 -1
```

Copy the output and set it as the value of:

```php
$cfg['blowfish_secret'] = 'your_random_string_here';
```

### 🔸 Set permissions

Allows Apache to access phpMyAdmin.

```bash
chown -Rv apache:apache /var/www/html/phpmyadmin/
```

### 🔸 Restart services

To apply changes for web and DB services.

```bash
systemctl restart httpd.service
systemctl restart mysqld.service
```

### 🔸 Access via browser

```
http://<server-ip>/phpmyadmin
```

### 🔐 Login Credentials

- **Username:** root
- **Password:** Your MySQL root password

---

## ✅ Final Checklist

| Component  | Test Command / URL                            |
| ---------- | --------------------------------------------- |
| PHP        | `php -v` and `http://<server-ip>/phpinfo.php` |
| MySQL      | `mysql -u root -p`                            |
| phpMyAdmin | `http://<server-ip>/phpmyadmin`               |

---

---

 by - [Anuj-Rawal](https://www.linkedin.com/in/anuj-r-70b364310/)
