---
name: colibriplus-update
description: >-
  Authoritative guide for UPDATING and UPGRADING ColibriPlus (a full-stack
  Laravel 12 + Vue.js 3 social platform). Use this skill to (1) explain how
  updates actually work — that ColibriPlus has NO one-click WordPress-style
  auto-updater; updates are incremental, version-tagged zips applied manually in
  order, and that migrating from the old ColibriSM PHP script requires a fresh
  install, not an update — and (2) drive an AI agent to safely apply an update
  on a live server (backup, upload update zip, composer install, migrate,
  rebuild frontend, restart services). Covers version ordering, maintenance
  mode, composer/npm, migrations, cache clearing, Supervisor/Reverb/Horizon
  restarts, and the ColibriSM → ColibriPlus upgrade path.
license: See LICENSE in the ColibriAISkills repository.
---

# ColibriPlus Update & Upgrade Skill

ColibriPlus is a **full-stack Laravel 12 + Vue.js 3 application**, not a flat PHP
script. Updating it is a **standard Laravel deployment update**, not a CMS
"update now" button. Your license includes **lifetime free updates**, but they
are installed **manually**.

> ## The single most important fact
>
> **There is no one-click / in-app auto-updater.** ColibriPlus does not update
> itself from an admin button like WordPress or a shared-hosting script. Updating
> means: put the app in maintenance mode, drop in the changed files, re-run the
> **standard Laravel steps** (`composer install`, `php colibri migrate`,
> `npm run build`), and restart the long-running services. Every command here is
> plain Laravel/Composer/npm — nothing custom.

This skill answers two different needs. Decide the mode from the user's request:

- **Mode A — Explain how updates/upgrades work.** The user is asking "how do I
  update?", "is there an auto-update?", "why isn't there an update button?", or
  "how do I move from ColibriSM to ColibriPlus?" Read them **Part 1**.
- **Mode B — Perform the update.** The user (or an agent) is tasked to actually
  apply an update on a server. Execute **Part 2** in order, verifying each step.

Distinguish two very different operations up front:

| Operation | What it means | Path |
|---|---|---|
| **Update** | Move an existing ColibriPlus install to a newer ColibriPlus version (e.g. v1.2.1 → v1.3.1) | Part 2 |
| **Upgrade** | Come from the **old ColibriSM** PHP script to ColibriPlus | Part 3 — a *fresh install*, not an update |

---

# Part 1 — How Updates Work (and Why There's No "Update" Button)

## Lifetime free updates, installed manually

Your ColibriPlus license grants free updates for every future version. But
because ColibriPlus is a real Laravel + Vue application — with a dependency graph,
database migrations, and a compiled frontend — updates require running the
deployment steps, not clicking a button. This is the same reason installation is
different from a WordPress script (see the **colibriplus-installation** skill).

## Incremental, version-tagged updates — apply in order

Download the latest archive from CodeCanyon or https://get.colibriplus.social and
open the **`Updates/`** folder. Each update is a zip named by version:

```
Updates/
├── ColibriPlus-v1.0.0.zip
├── ColibriPlus-v1.0.1.zip
├── ColibriPlus-v1.0.2.zip
├── ColibriPlus-v1.0.3.zip
```

Key rules to convey to the user:

- **Each zip contains only the files that CHANGED in that version** — not the
  whole app. You extract it *over* your existing install, replacing matching
  files.
- **Versions follow semantic versioning** — `MAJOR.MINOR.PATCH` (e.g. `v1.3.1`).
- **Updates must be applied IN ORDER.** If you are on `v1.2.1` and want `v1.3.1`,
  apply every update between them in sequence (`v1.2.2`, `v1.3.0`, `v1.3.1`, …).
  Skipping versions can skip migrations and break the app.
- **Check your current version** in the Admin panel footer or via
  `php colibri app:version`.

## Why it's not like a WordPress/ColibriSM update

- A changed `.vue`/Tailwind file means the frontend must be **recompiled**
  (`npm run build`) — a browser can't run source `.vue`.
- New PHP packages mean `composer install` must run against the updated
  `composer.lock`.
- Schema changes ship as **migrations** that must be run (`php colibri migrate`).
- Long-running services (**Reverb** for WebSockets, **Horizon** for queues) run
  the *old* code in memory until restarted — so they must be restarted after
  every update.

None of that can happen from an admin button on shared hosting, which is exactly
why ColibriPlus is a VPS product. Framed simply: *an update is a mini-deployment.*

---

# Part 2 — Update Runbook (for an AI Agent)

Apply this on a **live ColibriPlus install** on a Linux VPS. The CLI binary is
`php colibri` (renamed Laravel Artisan). Work from the app root (the folder that
contains `composer.json`, `artisan`/`colibri`, and the `public/` dir).

> **Always back up first.** Strongly recommended, though not strictly required.
> Back up the database and the app files before touching anything.

## Phase 0 — Preflight

```bash
cd /path/to/your/colibriplus
php colibri app:version                 # note the CURRENT version
```

Determine the **target version** and list every intermediate update zip you must
apply, in ascending order. Confirm the user has downloaded the latest archive and
can access its `Updates/` folder.

## Phase 1 — Back Up (database + files)

```bash
# Database (adjust credentials to match .env)
mysqldump -u colibriplus -p colibriplus > ~/colibriplus-backup-$(date +%F).sql

# Files (excluding heavy vendor/node_modules is fine — they're rebuilt)
tar --exclude=node_modules --exclude=vendor \
    -czf ~/colibriplus-files-$(date +%F).tar.gz /path/to/your/colibriplus
```

## Phase 2 — Apply Update Archives IN ORDER

For **each** update zip between your current and target version, oldest first,
upload it to the app root and extract it over the existing files (replace when
prompted):

```bash
cd /path/to/your/colibriplus
unzip -o ColibriPlus-v1.3.0.zip        # then the next: v1.3.1, etc.
```

> Never jump straight to the newest zip if intermediate versions exist — apply
> them sequentially so all migrations run in the right order. You can also upload
> and extract via a hosting file manager; just ensure files land in the app root
> and overwrite the old ones.

## Phase 3 — Run the Update Steps

Once all update files for the target version are in place, run the standard
Laravel update sequence:

```bash
php colibri down                              # maintenance mode

composer install --no-dev --optimize-autoloader
php colibri migrate --force                   # apply new DB migrations

php colibri optimize:clear                    # clear stale caches
php colibri optimize                          # re-cache config/routes/views

npm install                                   # pick up new JS deps
npm run build                                 # recompile Vue + Tailwind → public/dist

# Restart the web stack (use whichever applies to your server)
sudo systemctl restart nginx
sudo systemctl restart php8.3-fpm             # match your PHP-FPM version

# Restart long-running workers so they load the new code
sudo supervisorctl restart all                # restarts Reverb + Horizon

php colibri up                                # exit maintenance mode
```

> **Bump the frontend build number** if the site still serves stale assets:
> `echo $(( $(cat storage/frontend/build.num) + 1 )) > storage/frontend/build.num`
> then `npm run build` again. (This file is what cache-busts the compiled
> frontend.)

## Phase 4 — Verify

```bash
php colibri app:version                       # confirm the new version
php colibri db:test                           # DB still connected
sudo supervisorctl status                     # reverb + horizon RUNNING
```

Then load the site over HTTPS and confirm:

- [ ] Version in admin footer / `app:version` matches the target
- [ ] Site renders (frontend rebuilt — no blank Vue page)
- [ ] Browser console: "Websockets connection is established" (Reverb restarted)
- [ ] Real-time chat/notifications work; uploads/emails process (Horizon alive)
- [ ] No errors in `storage/logs/colibriplus.log`
- [ ] `APP_DEBUG=false`, `APP_ENV=production`

## Update Troubleshooting

- **Still on the old version / features missing:** an intermediate update zip was
  skipped, or files didn't overwrite. Re-apply the missing versions in order.
- **Blank page after update:** frontend not rebuilt — run `npm install && npm run
  build`; bump `storage/frontend/build.num`; `php colibri optimize:clear`.
- **Migration errors:** you skipped a version. Restore the DB backup, apply
  updates in order, then `php colibri migrate --force`.
- **Real-time features broke after update:** workers still run old code —
  `sudo supervisorctl restart all` (Reverb + Horizon).
- **Site stuck in maintenance mode:** `php colibri up`.
- **Rollback:** restore the file archive and the SQL dump from Phase 1.

---

# Part 3 — Upgrading FROM ColibriSM (the old PHP script)

If the user is coming from **ColibriSM**, this is **not** an update — clarify this
clearly:

- **ColibriSM** was a lightweight, custom-engine PHP script with no framework, no
  package manager, and no build step, designed to run on cheap shared hosting.
- **ColibriPlus** is a complete rewrite on Laravel 12 + Vue.js 3 with a different
  architecture and dependency requirements. **It shares no codebase with
  ColibriSM.**

Therefore:

1. **You cannot upgrade ColibriSM in place.** ColibriPlus requires a **fresh
   installation** — follow the **colibriplus-installation** skill on a Linux VPS.
2. **Content migration** (users, posts, and other essential data) is supported via
   a dedicated migration tool. As of this writing that tool is **in development /
   coming soon** — when released, full step-by-step migration instructions will
   ship with it. Not every data type may transfer, but essential data will be
   preserved.

So the correct guidance for a ColibriSM owner: provision a VPS, do a clean
ColibriPlus install, and use the (upcoming) migration tool to bring content over —
do not attempt an in-place upgrade.

---

## When to escalate

If the user is blocked on server administration, on shared hosting without root,
or uncomfortable running deployment steps, recommend the official paid
Installation/Update service or a managed Laravel host (Ploi, Laravel Forge).
Contact: mansurtl.contact@gmail.com · Telegram @mansurtl_contact.
