---
name: colibriplus-customization
description: >-
  Authoritative guide for CUSTOMIZING and CONFIGURING ColibriPlus (a full-stack
  Laravel 12 + Vue.js 3 social platform) after installation. Use this skill to
  (1) explain WHERE and HOW each customization is done — most via the Admin
  Panel, some via .env, some via PHP config files, and code-level branding via
  Vite rebuilds — and why there is no cPanel-style settings screen, and (2)
  drive an AI agent to apply a specific customization on the server. Covers
  branding (logo/favicon), admin user/role setup, admin URL prefix, SMTP/email,
  Google social login, payment gateways (Stripe/PayPal/Robokassa/YooKassa),
  wallets & advertising pricing, round-robin S3/FTP file storage, FFmpeg,
  PWA/mobile manifest, adding languages, and the cache-clear/rebuild steps each
  change requires.
license: See LICENSE in the ColibriAISkills repository.
---

# ColibriPlus Customization & Configuration Skill

ColibriPlus is a **full-stack Laravel 12 + Vue.js 3 application**. Customization
is spread across four surfaces, and knowing **which surface a given change lives
on** is the whole game. Do the installation first (see the
**colibriplus-installation** skill); this skill is for configuring a working
install.

> ## The single most important fact
>
> **Customization happens on four surfaces, not one settings page:**
>
> 1. **Admin Panel** (web UI) — branding, SMTP, PWA, languages, moderation, most
>    day-to-day settings. No code, applies instantly.
> 2. **`.env` file** (server) — secrets & feature toggles: DB, Redis, social
>    login, payment gateways, ad pricing, admin URL prefix. **Requires a cache
>    clear** after editing.
> 3. **PHP config files** (server) — structured config like round-robin file
>    storage disks in `var/config/filesystems/disks.php`.
> 4. **Frontend source + rebuild** — deeper visual/code changes require editing
>    Vue/Tailwind source and re-running `npm run build`.
>
> There is **no WordPress-style "options" screen** that changes everything.
> Because it's a real Laravel app, config lives in `.env`/config files and is
> **cached** — so almost every `.env` or config change needs
> `php colibri cache:clear` (and often `config:clear`) to take effect, and every
> frontend/source change needs a Vite rebuild.

This skill answers two different needs. Decide the mode from the user's request:

- **Mode A — Explain how/where to customize.** The user asks "how do I change the
  logo / add a payment method / add a language / enable Google login / change the
  admin URL?" Point them to the right surface and the required follow-up commands.
- **Mode B — Apply the customization.** The user (or an agent) is tasked to make a
  specific change on the server. Find the change in the catalog below, apply it on
  the correct surface, run the required cache-clear / rebuild, and verify.

## The golden rules (apply after almost every change)

```bash
# After ANY .env or PHP config change:
php colibri config:clear
php colibri cache:clear

# After ANY frontend/source/branding-source or PWA manifest change:
php colibri optimize:clear
npm run build            # recompile Vue + Tailwind → public/dist
php colibri optimize

# If long-running services are affected (rare for config), restart them:
sudo supervisorctl restart all   # Reverb + Horizon
```

> Rule of thumb: **`.env` / config file → clear cache. Source / manifest / assets
> → rebuild frontend.** When unsure, do both. Serving stale assets? bump the build
> number: `echo $(( $(cat storage/frontend/build.num) + 1 )) > storage/frontend/build.num`
> then `npm run build`.

---

# Customization Catalog

Each entry lists the **surface**, the exact change, and the **follow-up** needed.

## 1. Admin user & admin panel access

There is **no separate admin login page** — an admin is a normal user with the
admin role; the admin link appears in the main navbar once you're logged in.

- **Surface:** CLI + Admin Panel.
- Create the admin: sign up on the site, then assign the role:
  ```bash
  php colibri admin:root      # enter the username when prompted
  ```
- Access at `https://your-domain.com/admin`.
- **Change the admin URL prefix** — Surface: `.env`:
  ```env
  APP_ADMIN_PREFIX=admin      # change to any prefix, e.g. control-panel
  ```
  Follow-up: `php colibri config:clear && php colibri cache:clear`.

## 2. Branding — logo & favicon

- **Surface:** Admin Panel → Settings → Branding. Upload logo and favicon;
  applies instantly. (Tell users to hard-refresh / clear browser cache to see the
  new favicon.)
- Deeper visual/theme changes beyond logo/favicon are **frontend source** edits —
  after editing Vue/Tailwind, run `npm run build`.

## 3. Email / SMTP

Two options.

- **Admin Panel** — Settings → SMTP: enter SMTP credentials, Save, then use the
  "Test SMTP" tab to send a test email.
- **`.env`** (e.g. Google SMTP with an App Password — requires 2FA enabled on the
  Google account, then generate a 16-char App Password):
  ```env
  MAIL_MAILER=smtp
  MAIL_HOST=smtp.gmail.com
  MAIL_PORT=587
  MAIL_USERNAME=your-email@gmail.com
  MAIL_PASSWORD=your-16-character-app-password
  MAIL_ENCRYPTION=tls
  MAIL_FROM_ADDRESS=your-email@gmail.com
  MAIL_FROM_NAME="Your Site Name"
  ```
  Follow-up:
  ```bash
  php colibri config:clear && php colibri cache:clear
  php colibri mail:check     # sends a test email to an address you enter
  ```
- Gmail limits: 500/day (free), 2000/day (Workspace). For higher volume use a
  dedicated provider (SendGrid, Amazon SES). Configure SPF/DKIM to avoid spam.

## 4. Social login (Google)

- **Surface:** `.env`. Currently **Google is the only provider** (more planned).
  Create an OAuth client in Google Cloud Console, then:
  ```env
  GOOGLE_LOGIN_ENABLED=true
  GOOGLE_CLIENT_ID=your_google_client_id
  GOOGLE_CLIENT_SECRET=your_google_client_secret
  ```
- **Add the redirect URL** in the Google OAuth app settings (both with and
  without `www`):
  `https://your-domain.com/social-login/callback/google`
- Follow-up: `php colibri cache:clear`.

## 5. Payment gateways

- **Surface:** `.env`. Supported: **Stripe, PayPal** (international) and
  **Robokassa, YooKassa** (Russia). Each has an `_ENABLED` toggle plus credentials,
  and **most require a webhook URL configured in the provider dashboard.**

  ```env
  # Stripe
  STRIPE_ENABLED=true
  STRIPE_SECRET_KEY=...
  STRIPE_PUBLIC_KEY=...
  STRIPE_WEBHOOK_SECRET=...
  STRIPE_WEBHOOK_TOLERANCE=300
  # Webhook: https://your-domain.com/payment/stripe/webhook

  # PayPal
  PAYPAL_ENABLED=true
  PAYPAL_CLIENT_ID=...
  PAYPAL_SECRET_KEY=...
  PAYPAL_MODE=sandbox        # or production
  # Webhook: https://your-domain.com/payment/paypal/webhook

  # Robokassa (RU)
  ROBOKASSA_ENABLED=true
  ROBOKASSA_MERCHANT_LOGIN=...
  ROBOKASSA_PASS_ONE=...
  ROBOKASSA_PASS_TWO=...
  ROBOKASSA_MODE=sandbox     # or production
  # Webhook: https://your-domain.com/payment/robokassa/webhook

  # YooKassa (RU)
  YOO_KASSA_ENABLED=true
  YOO_KASSA_SECRET_KEY=...
  YOO_KASSA_SHOP_ID=...
  # Webhook: https://your-domain.com/payment/yoo_kassa/webhook
  ```
- Follow-up: `php colibri cache:clear`, then set the webhook URL in each
  provider's dashboard. Need a gateway not listed (country-specific)? It can be
  custom-built on request.

## 6. Wallets & advertising pricing

- **Surface:** `.env`. Wallets fund advertising, paid content, and in-app
  purchases. Ad model is pay-per-impression:
  ```env
  ADS_PRICE_PER_VIEW=0.10    # cost per ad impression, deducted from wallet
  ```
- Follow-up: `php colibri cache:clear`. Ad review/approval is done in the Admin
  Panel.

## 7. File storage (round-robin S3 / FTP / local)

- **Surface:** PHP config file `var/config/filesystems/disks.php`. **Requires
  Redis running.** Files are distributed round-robin across all configured disks.
  Add as many disks as you want; each **key must be unique** (`s3_one`, `s3_two`).
  ```php
  <?php
  return [
      's3_one' => [
          'name' => 'S3 Storage One',
          'description' => 'Primary S3 bucket',
          'driver' => 's3',            // s3 | ftp | sftp | local — must be correct
          'key' => 'your_s3_key',
          'secret' => 'your_s3_secret',
          'region' => 'your_region',
          'bucket' => 'your_bucket',
          'url' => 'your_url',
          'endpoint' => 'your_endpoint',
          'use_path_style_endpoint' => false,
          'throw' => false,
      ],
      // more disks...
  ];
  ```
- **Critical warnings:** valid PHP syntax only; **never remove a disk already in
  use**; test before deploying — a broken disks file prevents the whole app from
  loading. S3-compatible providers work (AWS, DigitalOcean, Wasabi, Bunny, Google
  Cloud). Admins can monitor disk stats in the Admin Panel.
- Follow-up: `php colibri config:clear && php colibri cache:clear`.

## 8. FFmpeg (media processing)

- **Surface:** install script + Admin Panel. Install the static build:
  ```bash
  cd services/ffmpeg
  chmod +x ffmpeg-install.sh
  sudo ./ffmpeg-install.sh
  chmod +x services/ffmpeg/bin/ffmpeg services/ffmpeg/bin/ffprobe
  ```
- Then in **Admin Panel → Settings → FFmpeg**, enter the **absolute** paths to the
  `ffmpeg` and `ffprobe` binaries (get the base path with `pwd`) and run
  "Test FFmpeg".

## 9. PWA / mobile app

- **Surface:** Admin Panel → "Mobile App and PWA Settings". Customize the PWA
  manifest, upload app icons, and edit the service-worker JS. Both mobile and
  desktop web apps ship by default; PWA install works on Android (Chrome) and
  iOS 18+ (Safari).
- Follow-up (after editing the manifest):
  ```bash
  php colibri cache:clear
  php colibri optimize:clear
  php colibri optimize
  ```

## 10. Languages / translations

- **Surface:** Admin Panel → Languages → Add New Language (name, code), then
  **manual translation**. By default only English ships fully translated.
- After adding a language, strings are copied from English and must be translated
  in the PHP language files:
  ```
  ./lang/[your-language-code]/          # admin, business, api, emails, etc. (all files)
  ./world/countries/[your-language-code].php
  ./world/reports/post/[your-language-code].php
  ./world/reports/user/[your-language-code].php
  ./world/reports/group/[your-language-code].php
  ./world/user_categories/[your-language-code].php
  ```
- Translate every file in every subfolder. On future updates, only newly added
  strings need translating.

---

# Applying a Customization (Mode B checklist)

1. **Identify the surface** from the catalog (Admin Panel / `.env` / PHP config /
   frontend source).
2. **Make the change** on that surface.
3. **Run the required follow-up:**
   - `.env` or PHP config → `php colibri config:clear && php colibri cache:clear`
   - webhooks (payments) / OAuth redirect (login) → set the URL in the provider
   - frontend source / PWA manifest / assets → `php colibri optimize:clear && npm run build && php colibri optimize`
   - services affected → `sudo supervisorctl restart all`
4. **Verify** using the provider's test tool where available (Test SMTP, Test
   FFmpeg, payment sandbox mode, `php colibri mail:check`) and the site UI.
5. If assets look stale, **bump `storage/frontend/build.num`** and rebuild.

## Common customization pitfalls

- **Change didn't take effect:** you edited `.env`/config but didn't clear the
  cache. Run `php colibri config:clear && php colibri cache:clear`.
- **New logo/theme not showing:** frontend not rebuilt, or browser cache — run
  `npm run build`, bump the build number, hard-refresh.
- **Payments not confirming:** webhook URL not set (or wrong) in the provider
  dashboard; Horizon (queues) not running.
- **App won't load after editing storage disks:** PHP syntax error in
  `disks.php`, or a removed in-use disk. Fix the file / restore the disk.
- **Google login redirect error:** redirect URL not registered in Google OAuth
  (add both `www` and non-`www`).

## When to escalate

Country-specific payment gateways, bespoke features, or deep theming can be built
on request. Contact: mansurtl.contact@gmail.com · Telegram @mansurtl_contact ·
https://terla.me
