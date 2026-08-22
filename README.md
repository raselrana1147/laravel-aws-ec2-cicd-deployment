# Laravel Project Deployment Guide (EC2 + GitHub Actions CI/CD)

A complete, beginner-friendly deployment guide for hosting a Laravel application on an AWS EC2 instance with Nginx, MySQL, and automated deployment via GitHub Actions. Every step below explains **what** the command does and **why** it's needed — so this doubles as a reference even if you're new to DevOps.

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

Before touching any commands, it helps to understand the overall picture of what we're building:

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

**In plain words:** You write code and push it to GitHub. GitHub automatically logs into your server and updates it with the new code, installs anything it needs, and refreshes the database — without you ever manually touching the server again after initial setup. Nginx is the "front door" that receives visitor requests; PHP-FPM is the engine that actually runs your Laravel code; MySQL is where your data (like the students table) lives.

---

## Prerequisites

- An AWS account with an EC2 instance (Ubuntu 22.04 LTS recommended) — this is your virtual server in the cloud
- A GitHub repository containing your Laravel project — this is where your code lives and where CI/CD is configured
- SSH access to your EC2 instance (a `.pem` key file, given to you by AWS when you launch the instance) — this is like a password file that lets you log into the server securely
- Basic familiarity with typing commands in a terminal — you don't need to be an expert, just comfortable copy-pasting and reading output

---

## Part 1 — EC2 Server Setup

### 1.1 Launch EC2 Instance

An EC2 instance is simply a virtual computer that AWS runs for you in a data center. When creating one, choose:

- **AMI (Amazon Machine Image): Ubuntu 22.04 LTS** — this is the operating system that will run on your virtual server, similar to installing Windows or macOS on a physical computer. Ubuntu is a popular, well-supported Linux distribution for servers.
- **Instance type: `t2.micro`** — this defines how much CPU/RAM your server gets. `t2.micro` is small and free-tier eligible, good enough for learning or small apps. Larger apps need bigger instance types.
- **Security Group** — think of this as a firewall. It controls which types of traffic are allowed to reach your server, and from where:

  | Type | Port | Source | Why |
  |---|---|---|---|
  | SSH | 22 | Your IP only | Lets *you* log in remotely to manage the server. Restricting to your IP (not "anywhere") stops random people on the internet from even trying to guess your password/key |
  | HTTP | 80 | Anywhere (0.0.0.0/0) | Lets any visitor's browser reach your website over regular (unencrypted) web traffic |
  | HTTPS | 443 | Anywhere (0.0.0.0/0) | Lets visitors reach your website securely (encrypted), needed if you add SSL later |

### 1.2 Connect to the Instance

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

**What this does:** `ssh` is the command used to remotely log into another computer over the network. `-i your-key.pem` tells it which private key file to use to prove your identity (instead of a password). `ubuntu@<EC2_PUBLIC_IP>` means "log in as the user `ubuntu`, on the server at this IP address." Once this succeeds, your terminal is now controlling the remote EC2 server directly — every command you type after this runs *on the server*, not on your own computer.

---

## Part 2 — Install Required Packages

```bash
sudo apt update && sudo apt upgrade -y
```
**What this does:** `apt` is Ubuntu's package manager — the tool used to install/update software. `apt update` refreshes the list of available software versions (it doesn't install anything yet, just checks what's new). `apt upgrade -y` then actually installs the latest versions of already-installed software. `sudo` means "run this as an administrator" — required because system-level changes need elevated permission. `-y` auto-answers "yes" to any confirmation prompts, so the command doesn't pause waiting for input.

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```
**What this does:** By default, Ubuntu's built-in software sources may only offer one specific PHP version, which might be older than what Laravel needs. `ppa:ondrej/php` is a trusted, widely-used third-party source ("PPA" = Personal Package Archive) that offers multiple PHP versions, including the newest ones. `software-properties-common` is a helper tool needed to add PPAs in the first place. After adding a new source, `apt update` refreshes the list again so it now includes packages from this new PPA too.

```bash
sudo apt install -y php8.2 php8.2-fpm php8.2-mysql php8.2-mbstring \
php8.2-xml php8.2-bcmath php8.2-curl php8.2-zip php8.2-gd \
unzip git curl nginx
```
**What this does:** Installs PHP version 8.2 specifically (not just "whatever PHP is default"), along with a set of add-on modules ("extensions") that Laravel needs, plus a few general tools. Here's what each one is actually for:

| Package | Plain-English Explanation |
|---|---|
| `php8.2` | The core PHP program itself — without this, nothing PHP-related works at all |
| `php8.2-fpm` | Short for "FastCGI Process Manager." PHP by itself can't talk to Nginx directly — FPM runs PHP in the background as a service that Nginx can send requests to. Think of it as the engine that actually executes your Laravel code when someone visits your site |
| `php8.2-mysql` | Lets PHP talk to MySQL databases. Without this, Laravel literally cannot connect to your database — you'd get a "driver not found" error |
| `php8.2-mbstring` | Handles text properly, especially non-English characters (accents, Bangla script, emojis, etc.). Laravel actually refuses to install via Composer without this extension present |
| `php8.2-xml` | Lets PHP read/write XML files. Composer and testing tools like PHPUnit rely on this internally, even if your app never directly touches XML |
| `php8.2-bcmath` | Enables precise math with very large or very precise decimal numbers (useful for money calculations). Some Laravel features and packages require it even if you don't use money math yourself |
| `php8.2-curl` | Lets your PHP code make outgoing web requests (e.g., calling another API). Laravel's built-in `Http::` helper depends on this |
| `php8.2-zip` | Lets PHP work with `.zip` files. Composer uses this to quickly unpack downloaded packages |
| `php8.2-gd` | Adds image-processing capability (resizing, cropping, thumbnails) — needed if your app handles image uploads |
| `unzip` | A general system tool (not PHP-specific) that Composer sometimes calls directly to extract files |
| `git` | Lets the server download code from GitHub repositories (`git clone`, `git pull`). Needed both for your initial setup and for CI/CD, which regularly pulls new code |
| `curl` | A command-line tool for making web requests, useful for testing (e.g., checking if your site responds) and used internally by the Composer installer |
| `nginx` | The actual web server software — the program that listens for visitor requests on the internet and decides how to respond (serve a file directly, or hand it off to PHP-FPM) |

> **Why version-pinned (`php8.2-fpm`) instead of unversioned (`php-fpm`)?** Without a version number, Ubuntu installs whatever the "default" PHP happens to be in its repositories — which can silently change between Ubuntu releases or PPA updates. Being explicit means you always know exactly which PHP version is running, which matters because Laravel has minimum version requirements.

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```
**What this does:** Composer is PHP's package manager — the tool that downloads and manages all the external libraries your Laravel project depends on (Laravel itself is installed via Composer). The first line downloads Composer's official installer script and immediately runs it using PHP, which produces a file called `composer.phar`. The second line moves that file into `/usr/local/bin/`, a folder that's already in the system's PATH — this simply means you can now type `composer` from anywhere in the terminal instead of needing to reference the full file path every time.

---

## Part 3 — Database Setup

Every Laravel app needs a database to store its data (students, users, etc.). You have two choices for **where** that database lives — choose one.

### Option A — AWS RDS (Managed MySQL) — Recommended for Production

RDS is AWS's "managed database" service — instead of installing MySQL yourself, AWS runs and maintains a dedicated database server for you (separate from your EC2 instance), and handles backups, security patches, and scaling automatically.

1. Go to **AWS Console → RDS → Create database → Engine: MySQL**
2. **Instance class: `db.t3.micro`** (free tier) or larger, depending on your needs — this determines the database server's CPU/RAM, similar to choosing an EC2 instance type
3. **Public access: No** — this is important. It means the database is *only* reachable from inside your AWS network (like your EC2 instance), never directly from the public internet. This is a major security improvement, since a database with no public exposure can't be attacked directly by outsiders
4. **VPC Security Group** — create or select one (e.g., `rds-sg`) and set its inbound rules to allow port `3306` (MySQL's default port) **only from your EC2 instance's own security group** — not from "anywhere." This means literally only your EC2 server is allowed to even attempt a connection
5. Once created, AWS gives you an **endpoint** — a long hostname like `mydb.xxxxx.us-east-1.rds.amazonaws.com`. This is the "address" your Laravel app will use to find the database, similar to a website domain but for a database server

**Important — a common point of confusion:** Creating an RDS instance does **not** automatically create a usable database inside it (unless you specifically typed a name into the optional "Initial database name" field during setup). The RDS instance is just the *server* — you still need to create an actual database (a named container for your tables) inside it:

```bash
mysql -h <RDS_ENDPOINT> -u admin -p -e "CREATE DATABASE laravel_db;"
```
**What this does:** `mysql` is the command-line client used to connect to any MySQL server. `-h <RDS_ENDPOINT>` says which server to connect to. `-u admin` says which username to log in as. `-p` tells it to prompt you for a password rather than typing it in plain text. `-e "..."` means "run this one SQL command and then exit" rather than opening an interactive session. `CREATE DATABASE laravel_db;` is the actual SQL instruction that creates a new, empty database named `laravel_db` — this is the container your Laravel tables (students, users, etc.) will live inside.

**Pros of RDS:** automated backups happen without you doing anything; AWS patches security updates for you; you can resize the database independently of your app server.
**Cons of RDS:** costs money separately from EC2; slightly more initial setup (security groups, VPC).

### Option B — MySQL Installed Directly on EC2 — Good for Learning/Small Projects

This means running the MySQL database *on the same server* as your Laravel app, instead of using a separate managed service.

```bash
sudo apt install mysql-server -y
sudo systemctl enable mysql
sudo systemctl start mysql
```
**What this does:** Installs the MySQL database software directly onto your EC2 server. `systemctl enable mysql` tells the server to automatically start MySQL every time the server reboots (so you don't have to manually start it after every restart). `systemctl start mysql` starts it right now, for the current session.

```bash
sudo mysql_secure_installation
```
**What this does:** Runs an interactive wizard that tightens MySQL's default security — it will ask you several yes/no questions:
- Set a root password → answer **Yes** and choose something strong
- Remove anonymous users → **Yes** (these are test accounts that ship by default and serve no purpose in production)
- Disallow root login remotely → **Yes** (since your app and database are on the same server, you never need to log in as root *from another machine*)
- Remove the test database → **Yes** (a sample database MySQL creates by default, safe to delete)
- Reload privilege tables → **Yes** (applies all the changes you just made immediately)

Create the database and a dedicated user for your app:

```bash
sudo mysql
```
**What this does:** Logs you into the MySQL command-line interface as the root/admin user. On Ubuntu, `sudo mysql` works without typing a MySQL password because MySQL is configured to trust anyone who is already an authenticated system administrator (`sudo`) on this specific server — this is called `auth_socket` authentication.

```sql
CREATE DATABASE laravel_db;
CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY 'YourStrongPassword!';
GRANT ALL PRIVILEGES ON laravel_db.* TO 'laravel_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
**What each line does:**
- `CREATE DATABASE laravel_db;` — creates an empty database (container) named `laravel_db`
- `CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY '...';` — creates a new MySQL login account specifically for your Laravel app, separate from the root account. `@'localhost'` restricts this user to only connect from the same machine — it can never be used to log in remotely, which limits damage if the password ever leaked
- `GRANT ALL PRIVILEGES ON laravel_db.* TO 'laravel_user'@'localhost';` — gives this new user full permission (create/read/update/delete tables) but **only within `laravel_db`** — it has no access to any other database on this server
- `FLUSH PRIVILEGES;` — tells MySQL to immediately apply the permission changes you just made, rather than waiting for the next restart
- `EXIT;` — leaves the MySQL prompt and returns you to the normal terminal

> **Why not just use the `root` MySQL user for everything?** If your Laravel app only ever connects as `laravel_user`, and that user can only touch `laravel_db`, then even in a worst-case scenario (like a security bug in your app), an attacker's access is contained to just this one database — not your entire MySQL server.

**Pros of local MySQL:** no separate AWS billing (it's "free" since you're already paying for the EC2 instance); simpler mental model since everything is in one place.
**Cons of local MySQL:** no automatic backups (you'd need to set up your own with `mysqldump` + cron); the database now competes with your app for the same CPU/RAM; if the EC2 instance goes down, both your app and database go down together.

> **Naming tip:** Avoid hyphens in database names (e.g., `my-db`) — MySQL requires awkward backtick-quoting around hyphenated names in every query. Use underscores instead (`my_db`).

---

## Part 4 — Clone and Configure the Laravel Project

```bash
cd /var/www
sudo git clone https://github.com/<your-username>/<your-repo>.git my-laravel-app
sudo chown -R ubuntu:www-data /var/www/my-laravel-app
cd my-laravel-app
```
**What this does:** `cd /var/www` moves into `/var/www`, the conventional folder on Linux servers where website files are stored. `git clone <url> my-laravel-app` downloads your entire project from GitHub into a new folder called `my-laravel-app`. `chown -R ubuntu:www-data` changes *ownership* of all these files — `ubuntu` (you, so you can edit files) as the owner, and `www-data` (the user Nginx runs as) as the group, so Nginx can also read/serve these files. `-R` means "apply recursively," i.e., to every file and subfolder inside.

### Install Dependencies

```bash
composer install --no-dev --optimize-autoloader
```
**What this does:** Reads your project's `composer.json` file (a list of every external library your Laravel app depends on) and downloads all of them into a `vendor/` folder. `--no-dev` skips packages only needed during development (like testing tools) — production doesn't need them, so this keeps things smaller and slightly more secure. `--optimize-autoloader` makes PHP's internal class-loading mechanism faster, which matters for production performance.

### Configure Environment

```bash
cp .env.example .env
php artisan key:generate
```
**What this does:** Laravel's real configuration lives in a file called `.env`, which is **never** committed to GitHub (it contains secrets like database passwords). `.env.example` is a template with the same structure but blank/placeholder values, which *is* committed to GitHub for reference. This command copies that template to create your actual `.env` file. `php artisan key:generate` then creates a unique, random encryption key and inserts it into `.env` as `APP_KEY` — Laravel uses this key internally to encrypt sessions, cookies, and other sensitive data. This should only be run **once** per environment (running it again would invalidate all existing encrypted data).

```bash
nano .env
```
**What this does:** Opens the `.env` file in `nano`, a simple terminal text editor. You'll edit these values:

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

**What each setting means:**
- `APP_NAME` — just the display name of your app, shown in some default Laravel pages/emails
- `APP_ENV=production` — tells Laravel it's running in a live environment, which changes some internal behaviors (like hiding detailed error pages from visitors)
- `APP_DEBUG=false` — **critical for security.** When `true`, Laravel shows visitors full error details (file paths, database queries, stack traces) if something breaks — extremely useful while developing locally, but a serious information leak on a live server
- `APP_URL` — the base URL of your site, used internally when Laravel generates links (e.g., in emails)
- `DB_CONNECTION=mysql` — tells Laravel which *type* of database driver to use. If this line is missing entirely, Laravel silently falls back to a default (often `sqlite`), which causes confusing "database file not found" errors even though you configured MySQL settings elsewhere
- `DB_HOST` — the address of your database server: `127.0.0.1` if MySQL is installed locally on this same EC2 instance, or your RDS endpoint if using AWS RDS
- `DB_PORT` — the network port MySQL listens on; `3306` is the standard default
- `DB_DATABASE` — the specific database name inside the MySQL server (the one you created in Part 3)
- `DB_USERNAME` / `DB_PASSWORD` — the login credentials for that database

> **Why `127.0.0.1` instead of `localhost` for local MySQL?** Both usually mean "this same machine," but PHP's MySQL driver sometimes interprets `localhost` as an instruction to connect via a Unix socket file instead of a normal network connection, which can cause confusing "can't connect" errors on some setups. Using the explicit IP `127.0.0.1` forces a standard TCP connection, avoiding that ambiguity.

### Run Migrations

```bash
php artisan migrate --force
```
**What this does:** Laravel "migrations" are version-controlled instructions (written in PHP) that create or modify database tables — e.g., the `students` table, `users` table, etc. Running `migrate` executes any migrations that haven't been applied yet, building out your actual database structure. `--force` is required because Laravel normally asks "are you sure you want to run this in production?" as a safety prompt — `--force` skips that prompt so it can run inside an automated script (like CI/CD) without waiting for a human to respond.

### Set File Permissions

```bash
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache
```
**What this does:** Laravel needs to *write* to two specific folders while running — `storage` (for logs, cached views, uploaded files) and `bootstrap/cache` (for compiled config/route caches). Since Nginx/PHP-FPM runs as the `www-data` user, this command gives `www-data` ownership of those folders so Laravel can actually write to them. `chmod -R 775` sets permissions so the owner and group can read/write/execute, while others can only read/execute — a reasonable balance between functionality and security for these specific folders.

### Cache for Production

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```
**What this does:** In normal development, Laravel re-reads config files, re-resolves routes, and re-compiles Blade templates on every single request — flexible for development, but slower. These three commands each compile their respective piece into a single optimized file, ahead of time, so production requests are noticeably faster. The trade-off: if you change `.env` or a route file later, you must re-run these commands (or `php artisan config:clear` etc.) for the changes to actually take effect — cached values otherwise take priority.

---

## Part 5 — Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/my-laravel-app
```
**What this does:** Opens a new, empty file where you'll define how Nginx should handle requests for this specific site. `/etc/nginx/sites-available/` is the conventional folder for storing *possible* site configurations (a server can host multiple sites, each with its own config file here).

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

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

**Line-by-line explanation:**
- `listen 80;` — tells Nginx to accept incoming connections on port 80 (the standard port for regular, unencrypted HTTP traffic)
- `server_name your_domain_or_ip;` — which domain name or IP address this configuration block applies to (replace with your actual EC2 IP or domain)
- `root /var/www/my-laravel-app/public;` — **very important:** this points to Laravel's `public` folder specifically, not the project root. Laravel's design intentionally keeps sensitive files (`.env`, application code) *outside* the public folder — only `public/` should ever be directly web-accessible
- `add_header X-Frame-Options "SAMEORIGIN";` — a security header that prevents your site from being embedded in an `<iframe>` on another website (protects against "clickjacking" attacks)
- `add_header X-Content-Type-Options "nosniff";` — tells browsers to strictly respect the declared file type rather than guessing, which prevents certain content-type-confusion attacks
- `index index.php;` — when someone visits a folder URL, Nginx looks for a file named `index.php` to serve by default
- `location / { try_files ... }` — this is the core of how Laravel routing works: for any incoming URL, Nginx first checks if it matches a real file or folder; if not, it passes the whole request to `index.php`, which is Laravel's actual entry point that handles all routing internally
- `location = /favicon.ico` / `robots.txt` — just quiets down log noise for these two very common, harmless requests browsers/crawlers make automatically
- `error_page 404 /index.php;` — even "not found" errors get routed through Laravel, so Laravel can show its own custom 404 page instead of a generic Nginx one
- `location ~ \.php$ { ... }` — this block specifically handles any request ending in `.php`. `fastcgi_pass unix:/run/php/php8.2-fpm.sock;` forwards it to PHP-FPM (the service from Part 2) via a Unix socket file — this is the actual handoff from "Nginx received a request" to "PHP is now executing your Laravel code"
- `location ~ /\.(?!well-known).* { deny all; }` — blocks direct web access to any file/folder starting with a dot (like `.env` or `.git`), **except** `.well-known` (which is needed for things like SSL certificate verification). This is an important security layer — even if file permissions were misconfigured, this rule stops sensitive dotfiles from being served over the web

```bash
sudo ln -s /etc/nginx/sites-available/my-laravel-app /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```
**What this does:** `ln -s ... sites-enabled/` creates a "symbolic link" (essentially a shortcut) of your config file into the `sites-enabled` folder — Nginx only actually uses configs that are linked here, so having a file in `sites-available` alone does nothing until it's linked. `rm -f .../default` removes Nginx's built-in placeholder site, so it doesn't conflict with yours. `nginx -t` tests your configuration file for syntax errors *before* applying it — always run this before reloading, so a typo doesn't take down your entire web server. `systemctl reload nginx` applies the new configuration without dropping any currently active connections (safer than a full restart).

Visit `http://<EC2_PUBLIC_IP>` in a browser to confirm the app is now live.

---

## Part 6 — Set Up GitHub Actions CI/CD

CI/CD stands for **Continuous Integration / Continuous Deployment** — in simple terms, it means "automatically test and deploy code whenever it changes," instead of manually SSH-ing in and running commands by hand every time.

### 6.1 Generate a Dedicated Deploy SSH Key

Run this **on your own local computer** (not on EC2):

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f github-actions-key
```
**What this does:** Creates a new SSH key **pair** — two files: `github-actions-key` (private) and `github-actions-key.pub` (public). Think of the public key as a lock you install on your server, and the private key as the one physical key that opens it. `-t ed25519` picks a modern, secure key algorithm. `-C "..."` is just a label/comment to help you remember what this key is for. `-f github-actions-key` sets the output filename. When prompted for a passphrase, leave it empty — a passphrase would require manual typing every time the key is used, which isn't possible in an automated GitHub Actions workflow.

- Add the **public key** (`github-actions-key.pub`) content to EC2's `~/.ssh/authorized_keys` file — this tells your server "anyone who has the matching private key is allowed to log in"
- Keep the **private key** (`github-actions-key`) to paste into GitHub as a secret in the next step

> **Why not just reuse your AWS `.pem` key for this?** If your GitHub secret were ever exposed (misconfigured workflow, compromised account, etc.), an attacker would only gain the specific access this dedicated key allows — and you can revoke/replace just this key without touching your main AWS access. Reusing your real `.pem` key would mean a leak gives full access to everything you can do on that server.

### 6.2 Add GitHub Repository Secrets

Go to your repo → **Settings → Secrets and variables → Actions → New repository secret**. "Secrets" are encrypted values GitHub stores securely and makes available to your workflow, without ever displaying them in logs.

| Secret Name | Value | Why It's Needed |
|---|---|---|
| `EC2_HOST` | EC2 public IP or domain | Tells the workflow which server to connect to |
| `EC2_USER` | `ubuntu` | Which username to log in as |
| `EC2_SSH_KEY` | Full content of the private key file | Proves the workflow is allowed to log in, without a password |
| `EC2_TARGET_DIR` | `/var/www/my-laravel-app` | Which folder on the server to deploy files into |

### 6.3 Create the Workflow File

Create this file at `.github/workflows/deploy.yml` inside your project:

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

**What this workflow does, step by step:**
- `on: push: branches: [ "main" ]` — this entire workflow only triggers automatically when code is pushed to the `main` branch. Pushes to other branches won't trigger a deployment
- `runs-on: ubuntu-latest` — GitHub runs this workflow on a **temporary** virtual machine that GitHub itself owns and manages (not your EC2, not your laptop) — it exists only for the duration of this workflow run
- **Checkout code** — downloads a fresh copy of your repository onto that temporary GitHub-owned machine, so it has the latest files to work with
- **Set up SSH Key** — loads your private deploy key into memory (an "SSH agent") so later steps can use it to authenticate, without ever writing the key to disk as a plain file
- **Add EC2 to known hosts** — normally, SSH shows an interactive "do you trust this server?" prompt the first time connecting to a new host; since this is automated, this step pre-approves your EC2 server's identity so the connection doesn't hang waiting for a human to type "yes"
- **Deploy files via rsync** — `rsync` efficiently copies files from the GitHub runner to your EC2 server, only transferring what's actually changed (faster than copying everything every time). The `--exclude` flags deliberately skip: `.env` (so your live secrets are never overwritten by a blank template), `/vendor` (dependencies are reinstalled fresh on the server instead of copied, since they can be OS/architecture-specific), `/storage` (contains logs and cached files specific to that server), and `/.git` (the git history isn't needed on the live server)
- **Run Laravel deployment commands** — opens an SSH connection to EC2 and runs a sequence of commands there: reinstalling Composer dependencies, running any new database migrations, rebuilding the config/route/view caches, fixing folder ownership (since freshly synced files may not have the right permissions), and finally reloading PHP-FPM so it picks up any code changes

**Two critical safety notes:**
- **`.env` is excluded from `rsync` on purpose.** It must already exist on the server (created manually once, back in Part 4) — this workflow will never create or overwrite it, which protects your live secrets from being wiped or replaced by a template.
- **Never add `php artisan key:generate --force` to this workflow.** That command should only ever be run once, manually, during initial setup. If it ran on every deploy, it would generate a brand-new encryption key each time — instantly invalidating every logged-in user's session and corrupting any previously encrypted data.

### 6.4 Configure Passwordless Sudo for PHP-FPM Reload Only

The last line of the workflow (`sudo systemctl reload php8.2-fpm`) needs `sudo`, but `sudo` normally pauses to ask for a password — which will never come in an automated, non-interactive SSH session, causing the workflow to hang or fail. The fix is to allow this **one specific command** to run without a password, without opening up broader admin access.

On EC2:
```bash
sudo visudo -f /etc/sudoers.d/deploy-permissions
```
**What this does:** `visudo` is the safe way to edit sudo permission rules — it checks your syntax before saving, preventing a typo from locking you out of `sudo` entirely. `-f /etc/sudoers.d/deploy-permissions` creates a new, separate rules file (rather than editing the main system file directly), which is cleaner and easier to remove later if needed.

Add exactly this line:
```
ubuntu ALL=(ALL) NOPASSWD: /bin/systemctl reload php8.2-fpm
```
**What this does:** Grants the `ubuntu` user permission to run *only* this exact command (`systemctl reload php8.2-fpm`) without being asked for a password. Every other `sudo` command still requires a password as normal — this rule is intentionally narrow, following the principle of granting the least access necessary.

### 6.5 Push and Verify

```bash
git add .github/workflows/deploy.yml
git commit -m "Add CI/CD workflow for Laravel deploy"
git push origin main
```
**What this does:** Adds the new workflow file to git's tracking, commits it with a descriptive message, and pushes it to GitHub. The moment this push completes, GitHub Actions notices the new workflow file and — since you just pushed to `main` — immediately triggers a deployment run.

Check the **Actions** tab on your GitHub repository page — you'll see the workflow running live, with each step shown as it completes. A green checkmark means it succeeded; a red X means something failed, and you can click into the failed step to see the exact error output.

---

## Part 7 — SSL Setup (Optional, Recommended for Production)

SSL/HTTPS encrypts traffic between visitors and your server, and is expected by modern browsers (which mark plain HTTP sites as "Not Secure"). This step requires you to already own a domain name pointed at your EC2's public IP (via a DNS "A record").

```bash
sudo nano /etc/nginx/sites-available/my-laravel-app
# change: server_name your_domain_or_ip; → server_name yourdomain.com;
```
**What this does:** Updates your Nginx config so it recognizes your actual domain name, not just the raw IP — required for the next step to work, since SSL certificates are issued per-domain.

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```
**What this does:** `certbot` is a free, automated tool from Let's Encrypt (a nonprofit certificate authority) that obtains and installs SSL certificates. The `python3-certbot-nginx` plugin lets it automatically edit your Nginx config for you, adding the correct HTTPS settings without you having to write them by hand. `-d yourdomain.com -d www.yourdomain.com` specifies which domain(s) the certificate should cover.

Update `.env` afterward:
```env
APP_URL=https://yourdomain.com
```
**Why:** Laravel uses `APP_URL` internally when generating absolute links (e.g., in password reset emails) — this should always match your site's real, current address.

---

## Security Checklist

- [ ] `APP_ENV=production` and `APP_DEBUG=false` on the live server — prevents leaking internal error details to visitors
- [ ] `.env` is listed in `.gitignore` and never committed to the repository — keeps secrets out of your git history
- [ ] Database uses a dedicated non-root user with least-privilege access — limits damage if credentials are ever exposed
- [ ] RDS (if used) has **Public access: No**, restricted to EC2's security group only — database isn't reachable from the open internet at all
- [ ] SSH port 22 restricted to your IP, not `0.0.0.0/0` — stops random internet scanners from even attempting to log in
- [ ] `.env`, `.git` blocked at the Nginx level (already included in the config above) — an extra layer of defense even if file permissions are ever misconfigured
- [ ] Dedicated GitHub Actions deploy key used — not your personal AWS `.pem` — limits blast radius of any potential leak
- [ ] Passwordless sudo scoped to only the exact command needed, never `NOPASSWD: ALL` — minimizes what an attacker could do even if the deploy key were compromised
- [ ] Strong, unique database password (not `12345678` or similar) — resists basic password-guessing attempts
- [ ] SSL/HTTPS enabled if a domain is in use — encrypts data in transit between visitors and your server

---

## Troubleshooting

| Error | Likely Cause | Fix |
|---|---|---|
| `Database file at path [...] does not exist. Connection: sqlite` | Missing `DB_CONNECTION=mysql` in `.env` — Laravel silently defaulted to SQLite | Add `DB_CONNECTION=mysql` explicitly, then run `php artisan config:clear` |
| `SQLSTATE[HY000] [1049] Unknown database` | The database name in `.env` was never actually created inside MySQL/RDS | Connect via `mysql -h <host> -u <user> -p` and run `CREATE DATABASE your_db;` |
| `ERROR 2003: Can't connect to MySQL server` | Security group is blocking traffic between EC2 and RDS | Add an inbound rule on RDS's security group allowing port 3306 from EC2's security group specifically |
| 502 Bad Gateway | PHP-FPM isn't running, or Nginx is pointing at the wrong socket path | Run `sudo systemctl status php8.2-fpm` to check it's active; confirm the socket path in your Nginx config matches the actual PHP-FPM version installed |
| Blank white page with no error shown | `APP_DEBUG=false` is correctly hiding a real underlying error from visitors | Check `storage/logs/laravel.log` on the server for the actual error details |
| Permission denied writing to storage/logs | `storage`/`bootstrap/cache` aren't owned by the `www-data` user | Re-run the `chown`/`chmod` commands from Part 4 |
| GitHub Actions workflow hangs indefinitely | A `sudo` command in the SSH script is waiting for a password that will never come | Set up scoped passwordless sudo for that exact command (Part 6.4) — never rely on interactive `sudo` inside CI/CD |

---

## Useful Commands Reference

```bash
# Check whether key services are currently running
sudo systemctl status nginx
sudo systemctl status php8.2-fpm
sudo systemctl status mysql

# Apply configuration changes without dropping active connections
sudo systemctl reload nginx
sudo systemctl reload php8.2-fpm

# Always check Nginx config for typos before reloading
sudo nginx -t

# Watch Laravel's error log live as it happens (Ctrl+C to stop watching)
tail -f storage/logs/laravel.log

# Clear all Laravel caches — useful first step when debugging odd behavior
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan cache:clear

# Rebuild caches for production performance after clearing them
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Confirm which PHP version is actually active
php -v

# Quickly test that a MySQL connection and credentials work
mysql -h <host> -u <user> -p -e "SHOW DATABASES;"
```

---

## Notes for Future Deployments

- This guide assumes a **single** EC2 instance. If traffic grows, consider a load balancer distributing requests across multiple EC2 instances, with sessions/cache moved to a shared store like Redis (since each instance would otherwise have its own separate, inconsistent local state).
- RDS is worth switching to once this becomes anything beyond a learning project — automated backups alone are usually worth the extra cost and setup.
- Treat this README as a living document — update it whenever the real deployment process changes, so it never goes stale or misleading for the next person (including future-you) who relies on it.
