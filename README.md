<div align="center">

# 🐘 php-mysql-starter

**A beginner-friendly [Claude Code Skill](https://code.claude.com/docs/en/skills) for building small, secure PHP + MySQL web apps.**

*From your first CRUD page all the way to a multi-role system with a status workflow — one consistent, secure pattern instead of slightly different code every session.*

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#-license)
[![PHP](https://img.shields.io/badge/PHP-7.4%2B-777BB4?logo=php&logoColor=white)](#)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql&logoColor=white)](#)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-D97757?logo=anthropic&logoColor=white)](https://code.claude.com/docs/en/skills)

</div>

---

## 📖 Table of Contents

- [🆕 Latest update](#-latest-update)
- [✨ What it covers](#-what-it-covers)
- [🗂️ Repo structure](#️-repo-structure)
- [🚀 Install](#-install)
- [💬 How to use it](#-how-to-use-it)
- [🎯 A note on scope](#-a-note-on-scope)
- [📄 License](#-license)

---

## 🆕 Latest update

> **This is the latest version of the skill I built.** Updated **July 2026** with the
> patterns I've been using in real projects, plus support for more AI agents:
>
> - 🤖 **Now works beyond Claude Code** — ships an `AGENTS.md` alongside `SKILL.md`, so
>   **Codex, Cursor, Gemini CLI, and GitHub Copilot CLI** can use it too.
> - 🗄️ **Safe live-schema changes** — auto-migrate a missing column, rename one without
>   breaking old code.
> - 🔄 **Live-updating lists** — polling + the event-delegation gotcha that breaks clicks
>   after a refresh.
> - 👥 **Richer workflows** — sub-stages as stage chips, and a sign-off / approval timeline.
> - 🧩 **JSON columns** — when they're OK, and how to read one whose shape has changed.

---

## ✨ What it covers

An [Agent Skill](https://code.claude.com/docs/en/skills) that teaches your AI coding agent the **conventions, boilerplate, and safety rules** for building small **PHP + MySQL** web apps — using plain HTML, CSS, and JavaScript. It ships both a `SKILL.md` and an `AGENTS.md`, so it works in **Claude Code, Codex, Cursor, Gemini CLI, and GitHub Copilot CLI** — see [Install](#-install).

It's for people **just starting to use an AI coding agent on a PHP project** — students, hobbyists, and devs shipping their first small app or system to shared hosting.

- 🔒 **Golden safety rules** built in by default — prepared statements (no SQL injection), hashed passwords, escaped output (no XSS), CSRF tokens on every state-changing action, credentials kept out of git.
- 🗄️ **Database creation & management** — create a database and user locally (phpMyAdmin / MySQL CLI) and on shared hosting (cPanel), grant privileges, back up / restore with `mysqldump`, and **change a table that already has live data safely** (auto-migrate a missing column, rename one without breaking old code).
- 📐 **Schema design** — choosing data types, primary/foreign keys, naming, a full worked example, and **when a JSON column is OK** (and how to read one whose shape has changed).
- 📋 **Copy-paste boilerplate** — connection file, login/register with `password_hash`, and a complete CRUD (Create-Read-Update-Delete) page set.
- 👥 **Roles & workflow** — a `role` column, `require_role()` guards, status columns, safe status transitions, **sub-stages within a status (stage chips)**, and a **sign-off / approval timeline** — the shape of an OMS-style app.
- 🎨 **UI patterns** — a reusable filterable/paginated data table, status badges, dashboard summary cards, flash messages (Post-Redirect-Get), a no-framework modal, and a **live-updating list** (polling + the event-delegation gotcha that bites everyone).
- 🩺 **Debugging** — a triage checklist for blank pages / HTTP 500s and the most common PHP + MySQL error messages with their fixes.
- 🤝 **How to work with your AI agent effectively** on a PHP project (what to paste, how to ask), so beginners get good results.

---

## 🗂️ Repo structure

```
php-mysql-starter/
├── SKILL.md                        # skill entry point — skill-aware agents read this first
├── AGENTS.md                       # same guidance for agents that read AGENTS.md/rules (Cursor, Codex, …)
└── references/
    ├── database.md                 # DB creation & management (local + cPanel), backups, safe live migrations
    ├── schema-design.md            # designing tables, keys, a worked schema, JSON columns
    ├── boilerplate.md              # db.php, auth, login/register, full CRUD
    ├── roles-and-workflow.md       # roles, require_role(), transitions, stage chips, sign-off timeline
    ├── ui-patterns.md              # tables, filters, pagination, badges, flash, CSRF, modals, live polling
    └── debugging.md                # blank page / 500 triage + common errors
```

`SKILL.md` and `AGENTS.md` carry the **same** rules — you don't pick one, they just let
different agents discover the skill their own way.

---

## 🚀 Install

The skill is just Markdown, so it works in any AI coding agent. Everywhere below, the
thing you install is the inner **`php-mysql-starter` folder** — the one with `SKILL.md`
and `AGENTS.md` directly inside it.

```bash
git clone https://github.com/GitRavz/php-mysql-starter.git
# the folder to install is: php-mysql-starter/php-mysql-starter
```

### Claude Code

Copy the folder into either location, then start a new session and run `/skills` to confirm:

- `~/.claude/skills/` — **personal**, available in all your projects, or
- `.claude/skills/` — **project-scoped**, committed with a repo so teammates get it too.

```bash
mkdir -p ~/.claude/skills
cp -r php-mysql-starter/php-mysql-starter ~/.claude/skills/php-mysql-starter
```

> ⚠️ Final path must be `~/.claude/skills/php-mysql-starter/SKILL.md` — not nested deeper.

### Codex, Copilot CLI, Gemini CLI (cross-runtime skills folder)

These agents discover skills from a shared `~/.agents/skills/` directory:

```bash
mkdir -p ~/.agents/skills
cp -r php-mysql-starter/php-mysql-starter ~/.agents/skills/php-mysql-starter
```

They read `SKILL.md` automatically. If your version doesn't support skills yet, use the
**Cursor / any agent** method below instead — the `AGENTS.md` covers the same ground.

### Cursor / Windsurf / any agent (via AGENTS.md or rules)

Agents without a skills folder read project context instead:

- **Simplest:** copy this project's `AGENTS.md` to your project root (Cursor, Codex,
  and others read a root `AGENTS.md` as standing instructions), and keep the
  `references/` folder next to it so the links resolve.
- **Cursor rules:** or point a rule at it — create `.cursor/rules/php-mysql-starter.mdc`
  whose body is “Follow the conventions in `AGENTS.md` and `references/*.md`,” and drop
  this folder into the project.

### Claude.ai (Pro / Max / Team / Enterprise, with code execution)

Zip the inner `php-mysql-starter` folder and upload it under
**Settings → Capabilities/Features → Skills**. (Custom skills are per-user.)

### Manual (last resort, any chat LLM)

Paste the contents of `SKILL.md` (or `AGENTS.md`) into your chat, then paste the specific
`references/*.md` file for the task at hand when the agent needs the detail.

---

## 💬 How to use it

Just work on your PHP project and talk naturally — the skill loads automatically when it's relevant. Triggers include things like:

- 💡 *"Help me start a PHP project with a login page."*
- 💡 *"Create a database and a users table for my app."*
- 💡 *"Add user roles so managers can approve and production can mark things done."*
- 💡 *"Track an order through pending → in production → released."*
- 💡 *"Add a form that saves to MySQL."* / *"Why is my page blank / showing a 500 error?"*

You can also nudge it: **"Use the php-mysql-starter skill for this."**

---

## 🎯 A note on scope

This skill teaches the **safe fundamentals of small systems** well rather than everything. Once these click, natural next steps are moving to PDO, splitting logic out of page files, and adopting a lightweight framework — intentionally out of scope here.

---

## 📄 License

[MIT](LICENSE) — free to use, modify, and share. Attribution appreciated.

<div align="center">

---

*Made to help beginners build real things with Claude Code.*
**Contributions and issues welcome!** 🚀

</div>
