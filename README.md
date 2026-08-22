# Laravel Project Deployment Guide (EC2 + GitHub Actions CI/CD)

Complete, repeatable deployment guide for hosting a Laravel application on an AWS EC2 instance with Nginx, MySQL, and automated deployment via GitHub Actions.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Part 1 — EC2 Server Setup](#part-1--ec2-server-setup)
4. [Part 2 — Install Required Packages](#part-2--install-required-packages)
5. [Part 3 — Database Setup](#part-3--database-setup)
6. [Part 4 — Clone and Configure the Laravel Project](#part-4--clone-and-configure-the-laravel-project)
7. [Part 5 — Configure Nginx](#part-5--configure-nginx)
8. [Part 6 — Set Up GitHub Actions CI/CD](#part-6--set-up-github-actions-cicd)
9. [Part 7 — SSL Setup (Optional)](#part-7--ssl-setup-optional)
10. [Security Checklist](#security-checklist)
11. [Troubleshooting](#troubleshooting)
12. [Useful Commands Reference](#useful-commands-reference)

---

## Architecture Overview

```
Developer pushes to GitHub (main branch)
              ↓
GitHub Actions triggers workflow
              ↓
SSH connects to EC2 → pulls code / syncs files
              ↓
Runs: composer install → migrate → cache
              ↓
Nginx + PHP-FPM serve the app
              ↓
App connects to MySQL (RDS or local on EC2)
```

---

## Prerequisites

- AWS account with an EC2 instance (Ubuntu 22.04 LTS recommended)
- A GitHub repository containing your Laravel project
- SSH access to your EC2 instance (`.pem` key from AWS)
- Basic familiarity with terminal commands

---

## Part 1 — EC2 Server Setup

### 1.1 Launch EC2 Instance

- AMI: **Ubuntu 22.04 LTS**
- Instance type: `t2.micro` (free tier) or larger depending on load
- Security Group inbound rules:
  | Type | Port | Source |
  |---|---|---|
  | SSH | 22 | Your IP only |
  | HTTP | 80 | Anywhere (0.0.0.0/0) |
  | HTTPS | 443 | Anywhere (0.0.0.0/0) |

### 1.2 Connect to the Instance

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

---

## Part 2 — Install Required Packages

```bash
sudo apt update && sudo apt upgrade -y

# Add PHP repository for version control
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# Install PHP 8.2 + required extensions
sudo apt install -y php8.2 php8.2-fpm php8.2-mysql php8.2-mbstring \
php8.2-xml php8.2-bcmath php8.2-curl php8.2-zip php8.2-gd \
unzip git curl nginx

# Install Composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

### What Each Package Does

| Package | Purpose |
|---|---|
| `php8.2` | Core PHP engine / CLI |
| `php8.2-fpm` | Runs PHP as a service; Nginx forwards `.php` requests here |
| `php8.2-mysql` | MySQL driver — required for Laravel DB connection |
| `php8.2-mbstring` | Multibyte string support (required by Laravel core) |
| `php8.2-xml` | XML parsing (needed by Composer, PHPUnit) |
| `php8.2-bcmath` | Precise math (used by some Laravel validation/packages) |
| `php8.2-curl` | Outgoing HTTP requests (`Http::` facade, package downloads) |
| `php8.2-zip` | Composer package extraction |
| `php8.2-gd` | Image processing (uploads, resizing) |
| `unzip` / `git` / `curl` | System utilities used by Composer and deployment scripts |
| `nginx` | Web server |

> **Note:** Always use version-pinned packages (`php8.2-*`) instead of unversioned (`php-*`) to avoid accidentally installing a different PHP version than expected.

---

## Part 3 — Database Setup

Choose **one** of the two options below.

### Option A — AWS RDS (Managed MySQL) — Recommended for Production

1. AWS Console → RDS → Create database → Engine: MySQL
2. Instance class: `db.t3.micro` (free tier) or appropriate size
3. **Public access: No** — only EC2 should reach it
4. VPC Security Group: create/select one (e.g. `rds-sg`) that allows inbound port `3306` **only from your EC2's security group**
5. Note the RDS **endpoint** once created (e.g. `mydb.xxxxx.us-east-1.rds.amazonaws.com`)

Create the actual database (RDS instance creation does NOT auto-create a usable database unless you set "Initial database name" during setup):

```bash
mysql -h <RDS_ENDPOINT> -u admin -p -e "CREATE DATABASE laravel_db;"
```

**Pros:** automated backups, easier scaling, patching handled by AWS
**Cons:** separate cost, slightly more setup

### Option B — MySQL Installed Directly on EC2 — Good for Learning/Small Projects

```bash
sudo apt install mysql-server -y
sudo systemctl enable mysql
sudo systemctl start mysql
sudo mysql_secure_installation
```

Create database and a dedicated (non-root) user:

```bash
sudo mysql
```
```sql
CREATE DATABASE laravel_db;
CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY 'YourStrongPassword!';
GRANT ALL PRIVILEGES ON laravel_db.* TO 'laravel_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Pros:** no extra AWS cost, simpler for learning
**Cons:** no automated backups, competes with app for server resources, single point of failure

> **Tip:** Never use hyphens in database names (e.g. `my-db`) — use underscores (`my_db`) to avoid constant backtick-quoting issues in SQL.

---

## Part 4 — Clone and Configure the Laravel Project

```bash
cd /var/www
sudo git clone https://github.com/<your-username>/<your-repo>.git my-laravel-app
sudo chown -R ubuntu:www-data /var/www/my-laravel-app
cd my-laravel-app
```

### Install Dependencies

```bash
composer install --no-dev --optimize-autoloader
```

### Configure Environment

```bash
cp .env.example .env
php artisan key:generate
nano .env
```

Set these values (adjust for RDS vs local MySQL from Part 3):

```env
APP_NAME=YourAppName
APP_ENV=production
APP_DEBUG=false
APP_URL=http://<EC2_PUBLIC_IP_OR_DOMAIN>

DB_CONNECTION=mysql
DB_HOST=127.0.0.1                # or your RDS endpoint
DB_PORT=3306
DB_DATABASE=laravel_db
DB_USERNAME=laravel_user         # or 'admin' if using RDS
DB_PASSWORD=YourStrongPassword!
```

> **Critical:** `APP_ENV=production` and `APP_DEBUG=false` are mandatory for any live server. `APP_DEBUG=true` leaks stack traces, file paths, and env values to anyone who triggers an error.

> **Important:** Use `127.0.0.1` rather than `localhost` for local MySQL — this forces a TCP connection and avoids socket-related connection issues with PHP's MySQL driver.

### Run Migrations

```bash
php artisan migrate --force
```

### Set File Permissions

```bash
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache
```

### Cache for Production

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

---

## Part 5 — Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/my-laravel-app
```

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;
    root /var/www/my-laravel-app/public;

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
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Block access to hidden files (.env, .git, etc.)
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/my-laravel-app /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Visit `http://<EC2_PUBLIC_IP>` to confirm the app is live.

---

## Part 6 — Set Up GitHub Actions CI/CD

### 6.1 Generate a Dedicated Deploy SSH Key

Run **locally** (not on EC2):

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f github-actions-key
```
Leave the passphrase empty.

- Add the **public key** (`github-actions-key.pub`) content to EC2's `~/.ssh/authorized_keys`
- Keep the **private key** (`github-actions-key`) to add as a GitHub secret

> **Why a separate key instead of your AWS `.pem`?** Limits the blast radius if the GitHub secret ever leaks — a dedicated deploy key can be revoked independently without affecting your personal AWS access.

### 6.2 Add GitHub Repository Secrets

Repo → **Settings → Secrets and variables → Actions → New repository secret**

| Secret Name | Value |
|---|---|
| `EC2_HOST` | EC2 public IP or domain |
| `EC2_USER` | `ubuntu` |
| `EC2_SSH_KEY` | Full content of the private key file |
| `EC2_TARGET_DIR` | `/var/www/my-laravel-app` |

### 6.3 Create the Workflow File

`.github/workflows/deploy.yml`:

```yaml
name: Deploy Laravel to EC2

on:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up SSH Key
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.EC2_SSH_KEY }}

      - name: Add EC2 to known hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.EC2_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy files via rsync
        run: |
          rsync -avz --exclude='.env' --exclude='/vendor' --exclude='/storage' --exclude='/.git' \
          ./ ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:${{ secrets.EC2_TARGET_DIR }}

      - name: Run Laravel deployment commands
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} << 'EOF'
            cd ${{ secrets.EC2_TARGET_DIR }}

            composer install --optimize-autoloader --no-dev

            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache

            chown -R www-data:www-data storage bootstrap/cache
            chmod -R 775 storage bootstrap/cache

            sudo systemctl reload php8.2-fpm
          EOF
```

### Important Notes on This Workflow

- **`.env` is excluded from rsync** — it must exist on the server already (created manually once during initial setup). It is never overwritten by CI/CD, protecting production secrets.
- **Never run `php artisan key:generate --force` in the CI/CD pipeline.** Generate the app key **once**, manually, during initial `.env` setup. Regenerating it on every deploy invalidates all active sessions and breaks any previously encrypted data.
- `chown`/`chmod` without `sudo` requires that the `ubuntu` user already has group ownership of `storage`/`bootstrap/cache` (set this up once — see Part 4). For the final `systemctl reload php8.2-fpm`, passwordless sudo must be configured for that specific command (see below).

### 6.4 Configure Passwordless Sudo for PHP-FPM Reload Only

On EC2:
```bash
sudo visudo -f /etc/sudoers.d/deploy-permissions
```
Add exactly this line (restrict to only what's needed):
```
ubuntu ALL=(ALL) NOPASSWD: /bin/systemctl reload php8.2-fpm
```

### 6.5 Push and Verify

```bash
git add .github/workflows/deploy.yml
git commit -m "Add CI/CD workflow for Laravel deploy"
git push origin main
```

Check the **Actions** tab in GitHub — a green checkmark confirms successful deployment.

---

## Part 7 — SSL Setup (Optional, Recommended for Production)

If you have a domain pointed at your EC2's public IP:

```bash
sudo nano /etc/nginx/sites-available/my-laravel-app
# change: server_name your_domain_or_ip; → server_name yourdomain.com;

sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Update `.env`:
```env
APP_URL=https://yourdomain.com
```

---

## Security Checklist

- [ ] `APP_ENV=production` and `APP_DEBUG=false` on the live server
- [ ] `.env` is in `.gitignore` and never committed to the repository
- [ ] Database uses a dedicated non-root user with least-privilege access (not `root`)
- [ ] RDS (if used) has **Public access: No**, restricted to EC2's security group only
- [ ] SSH port 22 restricted to your IP, not `0.0.0.0/0`
- [ ] `.env`, `.git` blocked at the Nginx level (already included in config above)
- [ ] Dedicated GitHub Actions deploy key used — not your personal AWS `.pem`
- [ ] Passwordless sudo scoped to only the exact command needed, never `NOPASSWD: ALL`
- [ ] Strong, unique database password (not `12345678` or similar)
- [ ] SSL/HTTPS enabled if a domain is in use

---

## Troubleshooting

| Error | Likely Cause | Fix |
|---|---|---|
| `Database file at path [...] does not exist. Connection: sqlite` | Missing `DB_CONNECTION=mysql` in `.env` | Add `DB_CONNECTION=mysql` explicitly, then `php artisan config:clear` |
| `SQLSTATE[HY000] [1049] Unknown database` | Database was never created inside MySQL/RDS | Connect via `mysql -h <host> -u <user> -p` and run `CREATE DATABASE your_db;` |
| `ERROR 2003: Can't connect to MySQL server` | Security group blocking EC2 → RDS traffic | Add inbound rule on RDS security group allowing port 3306 from EC2's security group |
| 502 Bad Gateway | PHP-FPM not running or wrong socket path in Nginx config | `sudo systemctl status php8.2-fpm`; confirm socket path matches Nginx config |
| Blank white page | `APP_DEBUG=false` hiding a real error | Check `storage/logs/laravel.log` on the server |
| Permission denied on storage/logs | Wrong ownership on `storage`/`bootstrap/cache` | Re-run the `chown`/`chmod` commands from Part 4 |
| GitHub Actions hangs on deployment commands | `sudo` prompting for a password in non-interactive SSH session | Configure scoped passwordless sudo (Part 6.4) — never use interactive-only `sudo` in CI/CD |

---

## Useful Commands Reference

```bash
# Check service status
sudo systemctl status nginx
sudo systemctl status php8.2-fpm
sudo systemctl status mysql

# Restart services
sudo systemctl reload nginx
sudo systemctl reload php8.2-fpm

# Test Nginx config before reloading
sudo nginx -t

# Laravel logs
tail -f storage/logs/laravel.log

# Clear all Laravel caches (useful when debugging)
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan cache:clear

# Re-cache for production after clearing
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Check current PHP version
php -v

# Test MySQL connection
mysql -h <host> -u <user> -p -e "SHOW DATABASES;"
```

---

## Notes for Future Deployments

- This guide assumes a single EC2 instance. For higher traffic, consider adding a load balancer with multiple EC2 instances behind it, and moving sessions/cache to Redis (shared state) instead of per-instance storage.
- RDS is recommended over local MySQL once this becomes anything beyond a learning project — automated backups alone are worth the switch.
- Keep this README updated as the deployment process evolves — treat it as a living document, not a one-time write-up.
