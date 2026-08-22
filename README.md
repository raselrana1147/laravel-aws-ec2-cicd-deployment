# Simple Laravel Deployment on AWS EC2

A quick, no-frills deployment process (LEMP stack: Linux, Nginx, MySQL, PHP) for a standard Ubuntu EC2 instance.

---

### 1. Update Server Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Check Your Available PHP Version

Skip guessing — check what your Ubuntu release already offers before installing anything:

```bash
apt-cache policy php
```

If it shows PHP 8.1 or newer, you're good to go with the commands below as-is. (If it shows nothing or an old version, you'll need the `ondrej/php` PPA first — ask if you hit this.)

### 3. Install Nginx

```bash
sudo apt install nginx -y
```
> Make sure your EC2 Security Group allows inbound **port 80 (HTTP)** and **443 (HTTPS)** — check this in the AWS Console under your instance's Security tab, not just once but if the site ever seems unreachable later.

### 4. Install and Configure MySQL

```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation
```

Create the database and a dedicated user:
```bash
sudo mysql
```
```sql
CREATE DATABASE laravel_app;
CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON laravel_app.* TO 'laravel_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 5. Install PHP and Required Extensions

```bash
sudo apt install -y php-fpm php-mysql php-xml php-mbstring php-curl php-zip php-bcmath php-gd unzip
```
> `php-zip` + `unzip` are both included on purpose — Composer needs at least one of them to extract packages, and it's easy to miss this and hit a confusing "zip extension...missing" error later.

### 6. Install Composer

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

### 7. Clone and Set Up the Laravel Project

```bash
cd /var/www
sudo git clone https://github.com/yourusername/your-repo.git myapp
sudo chown -R ubuntu:www-data myapp
cd myapp
```
> If the repo is **private**, a plain HTTPS clone will ask for a password — GitHub no longer accepts real account passwords for git. Use a Personal Access Token as the password, or set up an SSH key on the server and clone via `git@github.com:...` instead. Also: don't add `sudo` before `git clone` if you plan to use an SSH key you generated as your normal user — `sudo` looks for `root`'s key, not yours.

Install dependencies:
```bash
composer install --optimize-autoloader --no-dev
```
> If this fails with a PHP version compatibility error on a dev-only package (like `pest`/`paratest`), add `--ignore-platform-reqs` — dev dependencies aren't installed anyway when using `--no-dev`.

Set up the environment file:
```bash
cp .env.example .env
php artisan key:generate
nano .env
```
Update these lines:
```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_app
DB_USERNAME=laravel_user
DB_PASSWORD=StrongPassword123!
```

Generate the key and migrate:
```bash
php artisan migrate --force
```
> Note: `key:generate` only needs to run once, ever, for this environment — not on every future deploy.

### 8. Set Directory Permissions

```bash
sudo chown -R www-data:www-data /var/www/myapp
sudo chmod -R 775 /var/www/myapp/storage
sudo chmod -R 775 /var/www/myapp/bootstrap/cache
```

### 9. Configure the Nginx Server Block

```bash
sudo nano /etc/nginx/sites-available/myapp
```

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

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.x-fpm.sock; # check your actual version with: ls /run/php/
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

### If the site doesn't load in the browser

Check in this order:
1. Confirm the **exact Public IPv4 address** shown in the AWS Console (instance summary), not from an outside tool
2. Confirm the Security Group **attached to this instance** allows port 80 from `0.0.0.0/0`
3. Check `sudo ufw status` — if active, run `sudo ufw allow 80`
4. Confirm Nginx is actually listening: `sudo ss -tlnp | grep :80`

---

That's it — visit your EC2 instance's public IP and the app should be live.
