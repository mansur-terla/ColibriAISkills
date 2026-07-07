# ColibriPlus AI Skills

Public **AI Agent Skills** for [ColibriPlus](https://www.colibriplus.social) — a
full-stack **Laravel 12 + Vue.js 3** social media platform.

These skills exist because ColibriPlus is **not** a WordPress-style PHP script you
upload to cPanel. It is a real Laravel + Vue application that must be *deployed* on
a Linux VPS (Composer + npm, a compiled frontend, Redis, Reverb WebSockets,
Horizon queues, cron). Each skill teaches an AI assistant both to **explain** that
difference to a confused buyer and to **perform** the task correctly on a server.

## The skills

| Skill | What it does |
|---|---|
| [`colibriplus-installation`](skills/colibriplus-installation/SKILL.md) | Explains why ColibriPlus can't be installed on shared hosting/cPanel, and drives an end-to-end install on a Linux VPS. |
| [`colibriplus-update`](skills/colibriplus-update/SKILL.md) | Explains that there is no one-click auto-updater; drives a safe, in-order manual update. Also covers the ColibriSM → ColibriPlus upgrade path. |
| [`colibriplus-customization`](skills/colibriplus-customization/SKILL.md) | Explains the four customization surfaces (Admin Panel, `.env`, PHP config, frontend rebuild) and drives a specific config change (branding, SMTP, social login, payments, storage, PWA, languages…). |

Every skill has **two modes**:

- **Mode A — Explain.** Answer a user who linked the skill and is asking how it
  works / why their approach fails.
- **Mode B — Perform.** Act as (or instruct) an AI agent that must actually do the
  task on a server, step by step.

## Format

Each skill follows the **Agent Skills** convention: a folder named after the skill
containing a single self-contained `SKILL.md`, with YAML frontmatter (`name`,
`description`) that tells an agent what the skill is for and when to load it.

```
skills/
├── colibriplus-installation/SKILL.md
├── colibriplus-update/SKILL.md
└── colibriplus-customization/SKILL.md
```

A single `SKILL.md` is intentionally the whole skill, so it can be used **three
ways** (below) with no supporting files required.

## How to use these skills

### 1. As a knowledge file pasted into any AI chatbot

Open a `SKILL.md`, copy its full contents, and paste it into ChatGPT, Claude,
Gemini, Cursor, etc. Then ask your question ("how do I install ColibriPlus?") or
give the task ("install ColibriPlus on my Ubuntu VPS"). The frontmatter + body
give the model everything it needs.

### 2. As a public link an AI can read

Share a raw link to a `SKILL.md` and tell the assistant to read it. Options:

- **GitHub raw** — `https://raw.githubusercontent.com/<owner>/ColibriAISkills/master/skills/colibriplus-installation/SKILL.md`
- **Hosted on the docs site** — the ColibriPlus docs already publish a rendered
  copy at `https://docs.colibriplus.social/ai/installation-skill.html` (served as
  a static file, not a compiled page). You can publish the update and
  customization skills the same way (e.g. `/ai/update-skill.html`,
  `/ai/customization-skill.html`).

### 3. As an installed Agent Skill (Claude Code, Claude apps, Agent SDK)

Drop a skill folder where your agent loads skills from — e.g. a project's
`.claude/skills/` or a personal `~/.claude/skills/` — and the agent will pick it
up by its `name`/`description` automatically.

## Publishing the skills to the docs site (optional)

The docs repo serves raw skill files from `public/ai/` and its VitePress config
already excludes `**/ai/*.html` and `**/ai/*.md` from compilation, so angle-bracket
placeholders won't break the build. To publish the update/customization skills the
same way as the existing installation one, wrap each `SKILL.md` body in a minimal
HTML page inside a `<pre>` block and save it as
`ColibriDocs/public/ai/<skill>-skill.html`.

## License

See [`LICENSE`](LICENSE). ColibriPlus itself is sold under the Envato Market
Regular/Extended License.

## Author & support

Mansur Terla — creator of ColibriPlus.
Email: mansurtl.contact@gmail.com · Telegram: [@mansurtl_contact](https://t.me/mansurtl_contact) · Web: [terla.me](https://terla.me)

Prefer not to manage a VPS? A paid **Installation Service** ($139) and custom
development are available — see the docs' *Installation Service* page.
