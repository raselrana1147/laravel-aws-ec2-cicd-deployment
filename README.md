# Deploying a Laravel Application on AWS EC2

Deploying a Laravel application on an AWS EC2 instance requires setting up a complete **LEMP stack (Linux, Nginx, MySQL, PHP)** and configuring the server for your Laravel application.

This guide assumes you are using **Ubuntu** on your EC2 instance.

---

## 1. Update Server Packages

First, update the package index and upgrade the installed packages.

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 2. Install Nginx Web Server

### Package

`nginx`

Nginx is a high-performance web server. It receives HTTP requests and forwards PHP requests to PHP-FPM.

### Install Nginx

```bash
sudo apt install nginx -y
```

### Configure AWS Security Group

Make sure your EC2 Security Group allows inbound traffic on:

| Protocol | Port | Source      |
| -------- | ---: | ----------- |
| HTTP     |   80 | `0.0.0.0/0` |
| HTTPS    |  443 | `0.0.0.0/0` |

> Port 443 will be required later if you configure SSL/HTTPS.

---

## 3. Install and Configure MySQL

### Package

`mysql-server`

MySQL will store your Laravel application's database data.

### 3.1 Install MySQL

```bash
sudo apt install mysql-server -y
```

### 3.2 Secure MySQL Installation

Run the MySQL security script:

```bash
sudo mysql_secure_installation
```

Follow the prompts to remove insecure default settings such as anonymous users and the test database.

### 3.3 Create Database and User

Log into MySQL:

```bash
sudo mysql
```

Run the following SQL commands:

```sql
CREATE DATABASE laravel_app;

CREATE USER 'laravel_user'@'localhost'
IDENTIFIED BY 'StrongPassword123!';

GRANT ALL PRIVILEGES
ON laravel_app.*
TO 'laravel_user'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

> **Important:** Replace `StrongPassword123!` with a strong, unique password.

---

## 4. Install PHP and Required Extensions

Laravel requires PHP-FPM and several PHP extensions.

### Install PHP

```bash
sudo apt install php-fpm php-mysql php-xml php-mbstring php-curl php-zip php-bcmath -y
```

### Check PHP Version

```bash
php -v
```

Example:

```text
PHP 8.3.x
```

### Check PHP-FPM Status

```bash
sudo systemctl status php8.3-fpm
```

If your PHP version is different, replace `8.3` with your installed version.

---

## 5. Install Composer

Composer is the PHP dependency manager used by Laravel.

### Install Composer

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

### Verify Composer

```bash
composer --version
```

---

## 6. Set Up the Laravel Project

Navigate to the web root:

```bash
cd /var/www
```

### 6.1 Clone Your Project

Replace the repository URL with your actual GitHub repository.

```bash
sudo git clone https://github.com/yourusername/your-repo.git myapp
```

Enter the project directory:

```bash
cd /var/www/myapp
```

### 6.2 Install Laravel Dependencies

```bash
sudo composer install --optimize-autoloader --no-dev
```

### 6.3 Configure Environment Variables

Copy the example environment file:

```bash
sudo cp .env.example .env
```

Open the `.env` file:

```bash
sudo nano .env
```

Update your database configuration:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_app
DB_USERNAME=laravel_user
DB_PASSWORD=StrongPassword123!
```

Save the file.

### 6.4 Generate Application Key

```bash
sudo php artisan key:generate
```

### 6.5 Run Database Migrations

```bash
sudo php artisan migrate --force
```

---

## 7. Set Laravel Directory Permissions

Laravel needs write access to the `storage` and `bootstrap/cache` directories.

Set the application ownership:

```bash
sudo chown -R www-data:www-data /var/www/myapp
```

Set the required permissions:

```bash
sudo chmod -R 775 /var/www/myapp/storage
sudo chmod -R 775 /var/www/myapp/bootstrap/cache
```

`www-data` is the default user used by Nginx and PHP-FPM on Ubuntu.

---

## 8. Configure Nginx Server Block

Create an Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/myapp
```

Add the following configuration:

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;

    root /var/www/myapp/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    location = /robots.txt {
        access_log off;
        log_not_found off;
    }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

> **Important:** Make sure `php8.3-fpm.sock` matches the PHP version installed on your server.

You can check the available PHP-FPM socket with:

```bash
ls /var/run/php/
```

For example:

```text
php8.3-fpm.sock
```

---

## 9. Enable the Nginx Site

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
```

### Test Nginx Configuration

Always test the configuration before restarting Nginx:

```bash
sudo nginx -t
```

Expected result:

```text
syntax is ok
test is successful
```

### Restart Nginx

```bash
sudo systemctl restart nginx
```

---

## 10. Optional: Remove the Default Nginx Site

If the default Nginx page is still displayed, remove the default configuration:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Then test and restart Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## 11. Verify the Laravel Application

Open your browser and visit:

```text
http://your_ec2_public_ip
```

Or, if you configured a domain:

```text
http://your_domain.com
```

Your Laravel application should now be accessible.

---

## 12. Useful Service Commands

### Check Nginx

```bash
sudo systemctl status nginx
```

Start Nginx:

```bash
sudo systemctl start nginx
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

### Check MySQL

```bash
sudo systemctl status mysql
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

### Check PHP-FPM

```bash
sudo systemctl status php8.3-fpm
```

Restart PHP-FPM:

```bash
sudo systemctl restart php8.3-fpm
```

---

## 13. Useful Laravel Commands

After deployment, you may also need:

```bash
cd /var/www/myapp

sudo php artisan optimize
sudo php artisan config:clear
sudo php artisan cache:clear
sudo php artisan route:clear
sudo php artisan view:clear
```

For production optimization:

```bash
sudo php artisan optimize
```

---

## 14. Deployment Checklist

- [x] EC2 Ubuntu instance created
- [ ] Security Group allows port `80`
- [ ] Security Group allows port `443`
- [ ] Server packages updated
- [ ] Nginx installed
- [ ] MySQL installed
- [ ] MySQL database created
- [ ] MySQL user created
- [ ] PHP and required extensions installed
- [ ] Composer installed
- [ ] Laravel project cloned
- [ ] Composer dependencies installed
- [ ] `.env` configured
- [ ] Application key generated
- [ ] Database migrations completed
- [ ] Laravel directory permissions configured
- [ ] Nginx server block configured
- [ ] Nginx configuration tested
- [ ] Nginx restarted
- [ ] Application tested using EC2 public IP/domain

---

## Final Result

Once all steps are completed, your Laravel application will be running on your AWS EC2 instance using:

```text
AWS EC2
   │
   ▼
Ubuntu Linux
   │
   ├── Nginx
   │      │
   │      ▼
   │   Laravel / PHP-FPM
   │      │
   │      ▼
   │    MySQL
   │
   ▼
Public IP / Domain
```

Your application should be accessible through your EC2 public IP address or configured domain.
