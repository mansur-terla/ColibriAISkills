---
name: colibriplus-installation
description: >-
  Authoritative guide for installing ColibriPlus (a full-stack Laravel 12 +
  Vue.js 3 social media platform). Use this skill to (1) explain to a user WHY
  ColibriPlus is NOT a drop-in PHP script like WordPress/ColibriSM and cannot be
  installed by uploading files to cPanel/WHM/Apache shared hosting, and (2)
  drive an AI agent to perform a correct, end-to-end installation on a Linux
  VPS. Covers requirements, PHP/Node/Composer/MySQL/Redis, .env, migrations,
  Vite frontend build, Reverb WebSockets, Horizon queues, cron scheduler,
  Nginx + PHP-FPM, SSL, Supervisor, FFmpeg, and troubleshooting.
license: See LICENSE in the ColibriAISkills repository.
---

# ColibriPlus Installation Skill

ColibriPlus is a **full-stack web application** built on **Laravel 12 (PHP)** and
**Vue.js 3 + TailwindCSS + Livewire**. It is a Linux-first, VPS-first product. It
is **not** a "social script" you install by uploading files to cPanel/WHM and
opening `install.php`.

> ## The single most important fact
>
> **ColibriPlus is a plain, standard Laravel + Vue.js application. Nothing more.**
>
> There is **no custom engine, no invented installer, no proprietary deployment
> mechanism** created by the developer. Every step is a **standard Laravel and
> npm/Vite operation** that thousands of Laravel apps use identically:
> `composer install`, `npm install`, `.env`, `php artisan migrate` (exposed here
> as `php colibri`), `npm run build`, served behind Nginx + PHP-FPM,
> queues via Horizon, WebSockets via Laravel Reverb, scheduling via cron.
>
> **If you know how to deploy any Laravel + Vue application on Linux, you already
> know how to install ColibriPlus** — the distro (Ubuntu, Debian, CentOS, RHEL,
> Alma, Rocky, Arch, etc.) does not matter, only the standard PHP/Node/Redis/DB
> ecosystem does. When explaining or installing, lean on general Laravel + Vue
> knowledge; this skill only supplies the ColibriPlus-specific values (the
> `php colibri` binary name, Reverb port `13000`, the `/app` proxy path, the
> `storage/frontend/build.num` file, and the FFmpeg service path).

This skill answers two different needs. Decide which mode you are in from the
user's request:

- **Mode A — Explain the flow & the difference.** The user linked this skill and
  is asking how installation works, why their cPanel/Apache/shared-hosting
  attempt fails, or why "the docs don't work." Read them **Part 1**.
- **Mode B — Perform the installation.** The user (or an agent) is tasked to
  actually install ColibriPlus on a server using this skill. Execute **Part 2**
  step by step, top to bottom, verifying each step before moving on.

If the request is ambiguous, briefly explain Part 1 first, then offer to proceed
with Part 2.

---

# Part 1 — How Installation Works & Why It's Different

## The core misunderstanding

Many users come from **ColibriSM** (the predecessor) or from WordPress-style
scripts. Those are **flat PHP scripts**: a self-contained engine with no external
framework, no package manager, and no build step. You could unzip them into a
cPanel `public_html`, point Apache at the folder, run a web installer, and be
done. That model shaped their expectations — and it is exactly why they get stuck.

**ColibriPlus is a completely different class of software.** It is a modern
Laravel + Vue application with a real architecture, a dependency graph, a compiled
frontend, and multiple long-running background services. Uploading the files is
only the *first* of many steps, not the whole job.

> Key sentence to give the user:
> "ColibriSM was a PHP script you upload and run. ColibriPlus is a Laravel + Vue
> application you *deploy*. The files on disk are just source code — nothing runs
> until you install two sets of dependencies, build the frontend, configure the
> environment, run database migrations, and start the background services."

## Simple PHP script flow (what they *expect*)

```
Upload ZIP to cPanel  →  Extract into public_html  →  Apache serves index.php
→  Open /install in browser  →  Done
```

No Composer. No Node. No build. No queue worker. No WebSocket server. No Redis.
No CLI. This is the WordPress / ColibriSM mental model.

## Laravel full-stack flow (what ColibriPlus *actually* is)

```
VPS with root/sudo (NOT shared hosting)
   │
   ├─ System packages + PHP 8.3 extensions installed
   │
   ├─ Upload source  →  the `Source/` folder is the app root
   │
   ├─ composer install      → installs PHP/Laravel dependencies (incl. Reverb, Horizon)
   ├─ npm install           → installs JS/Vue/Tailwind build tooling
   │
   ├─ .env configured       → app identity, DB, Redis, Reverb, mail
   ├─ php colibri key:generate / migrate / db:seed / storage:link
   │
   ├─ npm run build         → Vite COMPILES Vue+Tailwind into public/dist
   │                          (browsers cannot run the raw .vue source)
   │
   ├─ Web server: Nginx  →  PHP-FPM  (document root = the `public/` dir, NOT app root)
   ├─ SSL (Certbot) — required, Reverb needs HTTPS/WSS
   │
   └─ Long-running services (kept alive by Supervisor):
        • Reverb   → WebSocket server (chat, notifications, live features)
        • Horizon  → Redis queue workers (video/audio processing, emails)
        • Cron     → `schedule:run` every minute (cleanup, maintenance)
```

## Everything here is standard Laravel + Vue — nothing custom

Every "ColibriPlus step" is a textbook Laravel/Vue step. There is nothing to
learn that isn't already Laravel or the npm/Vite ecosystem:

| ColibriPlus step | What it actually is (standard) | Laravel/Vue docs concept |
|---|---|---|
| `composer install` | Install PHP dependencies | Composer / Laravel install |
| `npm install` + `npm run build` | Compile Vue + Tailwind assets | Vite / Laravel Vite plugin |
| `.env` configuration | App/DB/cache/mail config | Laravel configuration & env |
| `php colibri key:generate` | `php artisan key:generate` | Laravel app key |
| `php colibri migrate --force` / `db:seed` | Run migrations & seeders | Laravel migrations |
| `php colibri storage:link` | Symlink `storage` → `public` | Laravel filesystem |
| Nginx root = `public/` + PHP-FPM | Serve a Laravel app | Laravel deployment |
| `php colibri reverb:start` | WebSocket server | Laravel Reverb (first-party) |
| `php colibri horizon` | Redis queue workers | Laravel Horizon (first-party) |
| `* * * * * … schedule:run` | Cron entry for the scheduler | Laravel Task Scheduling |
| Supervisor for Reverb/Horizon | Keep long-running processes alive | Laravel queues in production |

> The `php colibri` command **is** Laravel's Artisan CLI — it has simply been
> renamed. Any `php artisan <command>` you know maps 1:1 to `php colibri
> <command>`. Reverb and Horizon are official Laravel packages, not custom code.

So when guiding a user, frame it plainly: *this is a normal Laravel + Vue
deployment.* The reason a plain-PHP-script host fails is not that ColibriPlus is
exotic — it's that **any** Laravel + Vue app needs a real runtime.

## Why shared hosting / cPanel / WHM / Apache-only fails

Explain these concretely — they are the actual failure points users hit:

1. **No shell / no root.** Shared hosting rarely gives real SSH with sudo. You
   cannot run `composer install`, `npm run build`, or start services. The whole
   deployment depends on the CLI.
2. **No persistent processes.** cPanel kills long-running processes. ColibriPlus
   needs **Reverb** (WebSockets) and **Horizon** (queues) running *continuously*.
   Without them: no real-time chat/notifications, and uploads/emails never
   finish processing.
3. **No Redis.** Cache, sessions, queues, and round-robin file storage all
   require Redis. Most shared hosts don't offer it.
4. **No Node.js / no build step.** The frontend is **Vue + Tailwind source** that
   must be compiled by **Vite** into `public/dist`. A browser cannot execute
   `.vue` files. Shared hosts have no Node, so the frontend is never built and the
   site looks blank/broken.
5. **Wrong document root.** Apache/cPanel serve `public_html/` (the app root).
   Laravel's web root must be the **`public/`** subfolder. Pointing the server at
   the app root exposes `.env` and serves nothing usable.
6. **Missing PHP extensions.** Requires `gd, intl, mbstring, pdo_mysql, mysqli,
   bcmath, pcntl, zip, opcache, redis`. Shared hosts often lack several.
7. **WebSockets need a reverse proxy + HTTPS.** Reverb runs on its own port
   (13000) and Nginx must proxy `/app` to it with WebSocket upgrade headers over
   WSS. Apache-on-cPanel has no such configuration.

**Bottom line for the user:** the docs are correct — but they assume a **Linux
VPS/VDS with root access**, because that is the only environment where a
full-stack Laravel + Vue app can run. Installing on Apache/WHM shared hosting is
not "a harder version of the same thing"; it is a fundamentally unsupported
environment. If they don't want to manage a VPS, point them to the paid
**Installation Service** ($139) or a managed Laravel host (Ploi, Laravel Forge).

---

# Part 2 — Installation Runbook (for an AI Agent)

Execute these phases in order on **any Linux VPS** with sudo. The CLI binary is
`php colibri` (ColibriPlus's renamed Laravel Artisan). The application root is the
**`Source/`** folder from the downloaded archive.

> **Distro-agnostic:** the commands below use Ubuntu/Debian `apt` as the example,
> but ColibriPlus runs on **any Linux** — Debian, Ubuntu, CentOS/RHEL, Alma,
> Rocky, Fedora, Arch, openSUSE, Alpine, etc. Only the **package manager and
> package names differ**; the Laravel + Vue flow is identical everywhere. Adapt:
>
> | Distro family | Package manager | Example install |
> |---|---|---|
> | Debian / Ubuntu | `apt` | `sudo apt install php-fpm nginx redis-server` |
> | RHEL / CentOS / Alma / Rocky / Fedora | `dnf` / `yum` | `sudo dnf install php-fpm nginx redis` |
> | Arch / Manjaro | `pacman` | `sudo pacman -S php-fpm nginx redis` |
> | openSUSE | `zypper` | `sudo zypper install php-fpm nginx redis` |
> | Alpine | `apk` | `sudo apk add php-fpm nginx redis` |
>
> The required *software* is always the same: PHP 8.3+ with the listed
> extensions, Node.js 22+, Composer, a relational DB (MySQL/MariaDB/PostgreSQL),
> Redis, and Nginx. Install those with whatever your distro provides. Service and
> socket names also vary (e.g. `php8.3-fpm` on Debian vs `php-fpm` on RHEL) —
> check yours and substitute.

> Before you start, confirm the environment. **Abort and warn the user** if this
> is shared hosting / cPanel / WHM without root SSH — installation cannot succeed
> there (see Part 1). Confirm: a Linux VPS with sudo access, a domain pointed at
> the server IP, and minimum specs: 4 GB RAM, 10 GB SSD, multi-core CPU.

## Phase 0 — Preconditions & Requirements

Target versions: **PHP 8.3+**, **Node.js 22.x+** (docs' nvm example installs 24),
**MySQL 8.0+** (or PostgreSQL/MariaDB), **Redis 7.0+** (6.0+ acceptable),
**Composer 2.x**, Nginx.

Install system packages:

```bash
sudo apt update
sudo apt install -y --no-install-recommends \
    software-properties-common apt-utils libpq-dev libpng-dev openssl \
    libjpeg-dev libfreetype6-dev libwebp-dev libxpm-dev libgd-dev libzip-dev \
    libicu-dev zip unzip git libonig-dev libxml2-dev libsqlite3-dev libssl-dev \
    curl default-mysql-client
```

Install PHP + required extensions:

```bash
sudo apt install -y php php-cli php-fpm php-mysql php-gd php-intl \
    php-mbstring php-zip php-bcmath php-opcache php-curl php-xml php-redis
```

Verify all required extensions are present (must list every one):

```bash
php -m | grep -E "(gd|intl|mbstring|pdo_mysql|mysqli|bcmath|pcntl|zip|opcache|redis)"
```

## Phase 1 — Install Supporting Services

**Composer** (if missing):

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer
composer --version
```

**Node.js 22+** via nvm:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 24
node -v && npm -v
```

**MySQL** — install, secure, and create the database + user:

```bash
sudo apt install -y mysql-server
sudo mysql_secure_installation
```

```sql
CREATE DATABASE colibriplus CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'colibriplus'@'localhost' IDENTIFIED BY 'USE-A-STRONG-PASSWORD';
GRANT ALL PRIVILEGES ON colibriplus.* TO 'colibriplus'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Redis** — install, enable, verify (`redis-cli ping` → `PONG`):

```bash
sudo apt install -y redis-server
sudo systemctl enable redis && sudo systemctl start redis
redis-cli ping
```

**Nginx**:

```bash
sudo apt install -y nginx
sudo systemctl enable nginx && sudo systemctl start nginx
```

## Phase 2 — Upload Source & Set Permissions

The archive contains `Documentation/`, `Source/`, `Updates/`. Upload the contents
of **`Source/`** to the app root (e.g. `/var/www/colibriplus`). The web server's
root will later point at the `public/` subfolder inside it.

```bash
cd /path/to/your/colibriplus

sudo mkdir -p storage/app/public/uploads/posts/images
sudo mkdir -p storage/app/public/uploads/posts/videos
sudo mkdir -p storage/app/tmp/images
sudo mkdir -p storage/app/tmp/videos

sudo chown -R $USER:www-data .
chmod -R ug+rwx storage bootstrap/cache
```

## Phase 3 — Install Dependencies (both stacks)

```bash
composer install          # PHP/Laravel deps — installs Reverb & Horizon too
npm install               # JS/Vue/Tailwind build tooling
```

Do not skip either. `composer install` also installs the Reverb WebSocket server
and Horizon queue manager — they are not separate installs.

## Phase 4 — Environment Configuration

```bash
cp .env.example .env
```

Edit `.env`:

```bash
APP_NAME=ColibriPlus
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-domain.com
SANCTUM_STATEFUL_DOMAINS=your-domain.com,www.your-domain.com,localhost,127.0.0.1
ADMIN_EMAIL=admin@your-domain.com
BACKUP_EMAIL=admin@your-domain.com

# Database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=colibriplus
DB_USERNAME=colibriplus
DB_PASSWORD=USE-A-STRONG-PASSWORD

# Redis + queues/cache
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=null

# Reverb (WebSockets) — see Phase 9
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=13000
REVERB_HOST=your-domain.com
REVERB_PORT=443
REVERB_SCHEME=https
REVERB_APP_ID=generate-random
REVERB_APP_KEY=generate-random
REVERB_APP_SECRET=generate-random
VITE_REVERB_CONNECTION_STATUS=on
```

## Phase 5 — App Key, Migrations, Seed, Storage

```bash
php colibri key:generate
php colibri migrate --force
php colibri db:seed
php colibri livewire:publish --assets
php colibri vendor:publish --tag=log-viewer-assets --force
php colibri storage:link
```

If prompted to create the database, confirm with `Y`. Verify the DB connection:

```bash
php colibri db:test    # expect: OK. Your app is connected to database: colibriplus
```

## Phase 6 — Build the Frontend (mandatory)

The frontend is Vue + Tailwind source that **must** be compiled by Vite. Skipping
this leaves a blank/broken site. First create the build-number file:

```bash
mkdir -p storage/frontend
echo 1 > storage/frontend/build.num
npm run build            # outputs compiled assets to public/dist
```

## Phase 7 — Nginx + PHP-FPM (document root = `public/`)

Create a server block. **Root must point at the `public/` subfolder**, and `/app`
must reverse-proxy to Reverb on port 13000 with WebSocket upgrade headers. A full
reference config ships at `docker/nginx/conf.d/nginx.conf`. Minimal shape:

```nginx
server {
    listen 80;
    server_name your-domain.com;
    root /var/www/colibriplus/public;   # NOT the app root

    client_max_body_size 4G;
    index index.php index.html;
    charset utf-8;

    location ~* ^/storage/.*\.(php|phtml|php5|php7)$ { deny all; return 404; }
    location ~ /\.(env|git) { deny all; return 404; }

    location / { try_files $uri $uri/ /index.php?$query_string; }

    location /app {                       # Reverb WebSocket proxy
        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass http://localhost:13000;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;   # match your PHP version
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl restart php8.3-fpm
php colibri optimize
```

## Phase 8 — SSL (required)

Reverb needs HTTPS/WSS. Point the domain at the server IP, open ports 80 & 443,
then:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
sudo certbot renew --dry-run
```

## Phase 9 — Reverb WebSocket Server

Reverb (installed in Phase 3) powers chat, notifications, and live features.
Open its port and start it:

```bash
sudo ufw allow 13000
npm run build              # rebuild so the client has current Reverb config
php colibri reverb:start   # foreground test; use --debug for verbose output
```

For production, run it under Supervisor (Phase 11), not in the foreground.

## Phase 10 — Queues (Horizon)

Horizon (installed in Phase 3) processes background jobs (video/audio, emails)
via Redis. Ensure `QUEUE_CONNECTION=redis`, then:

```bash
php colibri horizon        # foreground test; run under Supervisor in production
```

## Phase 11 — Supervisor (keep Reverb & Horizon alive)

```bash
sudo apt install -y supervisor
```

`/etc/supervisor/conf.d/reverb.conf`:

```ini
[program:reverb]
directory=/var/www/colibriplus/
command=php colibri reverb:start
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/colibriplus/storage/logs/reverb.log
stopwaitsecs=3600
```

`/etc/supervisor/conf.d/horizon.conf`:

```ini
[program:horizon]
directory=/var/www/colibriplus/
command=php colibri horizon
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/colibriplus/storage/logs/horizon.log
stopwaitsecs=3600
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start reverb:* horizon:*
sudo supervisorctl status
```

> `numprocs=1` for both — neither Reverb nor Horizon supports multiple processes
> out of the box.

## Phase 12 — Cron Scheduler (single entry)

Laravel needs exactly one cron entry running the scheduler every minute:

```bash
crontab -e
```

```cron
* * * * * cd /var/www/colibriplus && php colibri schedule:run >> /dev/null 2>&1
```

Verify: `php colibri schedule:list`.

## Phase 13 — FFmpeg (media processing)

```bash
cd services/ffmpeg
chmod +x ffmpeg-install.sh
sudo ./ffmpeg-install.sh
chmod +x services/ffmpeg/bin/ffmpeg services/ffmpeg/bin/ffprobe
```

Then in Admin Panel → Settings → FFmpeg, enter the **absolute** paths to the
`ffmpeg` and `ffprobe` binaries (get the base path with `pwd`), and run
"Test FFmpeg".

## Phase 14 — Create the Admin User

The admin panel has no separate login — an admin is a normal user with the admin
role. After the site is up:

1. Sign up on the site with email, username, and password.
2. Assign the admin role from the CLI:

```bash
php colibri admin:root       # enter the username when prompted
```

Then log in and open `https://your-domain.com/admin` (prefix configurable via
`APP_ADMIN_PREFIX`).

## Phase 15 — Post-Install (admin-panel configuration)

These are done in the Admin Panel, not the CLI. For anything here, use the
**colibriplus-customization** skill:

- **SMTP / email** — Settings → SMTP (or use Google SMTP app password).
- **Social login** — Google OAuth, etc.
- **Payment gateways** — Stripe / PayPal / Robokassa / YooKassa via `.env`.
- **File storages** — round-robin S3/FTP disks (requires Redis) in
  `var/config/filesystems/disks.php`.
- **Logo / favicon**, **PWA / mobile app** settings, branding, languages.

## Final Verification Checklist

- [ ] `php -m` lists all required extensions
- [ ] `php colibri db:test` connects successfully
- [ ] `redis-cli ping` → `PONG`
- [ ] `npm run build` produced `public/dist`
- [ ] Site loads over **HTTPS** at the domain (Nginx root = `public/`)
- [ ] Browser console shows "Websockets connection is established"
- [ ] `supervisorctl status` shows `reverb` and `horizon` RUNNING
- [ ] `crontab -l` shows the scheduler entry
- [ ] FFmpeg test passes in admin panel
- [ ] Admin user created (`php colibri admin:root`) and `/admin` reachable
- [ ] `APP_DEBUG=false` and `APP_ENV=production`

---

## Alternative: Docker

If the user prefers containers, ColibriPlus ships a Docker setup (nginx, app,
MySQL, node, horizon, redis). From the app root:

```bash
docker-compose up -d          # or: docker-compose -f docker-compose.yml up -d --build
```

## Troubleshooting quick reference

- **500 error / blank page:** check `storage/logs/colibriplus.log`; fix
  permissions (`chmod -R ug+rwx storage bootstrap/cache`); `php colibri
  optimize:clear`.
- **Blank Vue UI:** the frontend wasn't built — run `npm install && npm run
  build`; confirm `public/dist` exists.
- **Real-time features dead:** Reverb not running or not proxied — check
  Supervisor status, the `/app` Nginx proxy, HTTPS/WSS, and port 13000.
- **Uploads/emails never finish:** Horizon not running or `QUEUE_CONNECTION`
  isn't `redis`.
- **DB errors:** verify `.env` credentials; `php colibri db:test`; `php colibri
  migrate --force`.
- **Uploads fail / too large:** raise PHP `upload_max_filesize`, `post_max_size`,
  `memory_limit`; ensure `php colibri storage:link` ran.

## When to escalate to the paid Installation Service

If the user is on shared hosting/cPanel with no root, doesn't want to manage a
VPS, or is blocked on server administration, recommend the official Installation
Service ($139) or a managed Laravel host (Ploi, Laravel Forge). Do not attempt to
force a full-stack Laravel deployment onto an unsupported shared-hosting
environment. Contact: mansurtl.contact@gmail.com · Telegram @mansurtl_contact.
