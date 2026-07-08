# php-mysql-basic — a Claude Code Skill

A beginner-friendly [Agent Skill](https://code.claude.com/docs/en/skills) that teaches
Claude Code the **conventions, boilerplate, and safety rules** for building small
**PHP + MySQL** web apps with plain HTML, CSS, and JavaScript.

It's for people who are **just starting to use Claude Code (or any SKILL.md-compatible
AI agent) on a PHP project** — students, hobbyists, and devs shipping their first small
app to shared hosting. Instead of getting slightly different code every session, Claude
follows one consistent, secure pattern.

## What it covers

- **Golden safety rules** built in by default — prepared statements (no SQL injection),
  hashed passwords, escaped output (no XSS), credentials kept out of git.
- **Database creation & management** — create a database and user locally (phpMyAdmin /
  MySQL CLI) and on shared hosting (cPanel), grant privileges, and back up / restore
  with `mysqldump`.
- **Schema design** — choosing data types, primary/foreign keys, naming, and a full
  worked example schema.
- **Copy-paste boilerplate** — connection file, login/register with `password_hash`,
  and a complete CRUD (Create-Read-Update-Delete) page set.
- **Debugging** — a triage checklist for blank pages / HTTP 500s and the most common
  PHP + MySQL error messages with their fixes.
- **How to work with Claude Code effectively** on a PHP project (what to paste, how to
  ask), so beginners get good results.

## Repo structure

```
php-mysql-basic/
├── SKILL.md                        # the skill entry point (Claude reads this first)
└── references/
    ├── database.md                 # DB creation & management (local + cPanel), backups
    ├── schema-design.md            # designing tables, keys, a worked schema
    ├── boilerplate.md              # db.php, auth, login/register, full CRUD
    └── debugging.md                # blank page / 500 triage + common errors
```

## Install

### Claude Code (recommended)

Skills live in one of two folders. Put the **`php-mysql-basic` folder** (the one with
`SKILL.md` directly inside it) into either:

- `~/.claude/skills/` — **personal**, available in all your projects, or
- `.claude/skills/` — **project-scoped**, committed with a specific repo so teammates
  get it too.

Quick way (personal install):

```bash
git clone https://github.com/GitRavz/php-mysql-basic-skill.git
mkdir -p ~/.claude/skills
cp -r php-mysql-basic-skill/php-mysql-basic ~/.claude/skills/
```

Then start a new Claude Code session and run `/skills` to confirm it loaded. The final
path must be `~/.claude/skills/php-mysql-basic/SKILL.md` — **not** nested a level deeper.

### Claude.ai (Pro / Max / Team / Enterprise, with code execution)

Zip the `php-mysql-basic` folder and upload it under **Settings → Capabilities/Features →
Skills**. (Custom skills are per-user.)

## How to use it

Just work on your PHP project and talk naturally — the skill loads automatically when
it's relevant. Triggers include things like:

- "Help me start a PHP project with a login page."
- "Create a database and a users table for my app."
- "Add a form that saves to MySQL."
- "Why is my page blank / showing a 500 error?"

You can also nudge it: *"Use the php-mysql-basic skill for this."*

## A note on scope

This skill deliberately teaches the **safe basics** well rather than everything. Once
the fundamentals click, natural next steps are CSRF tokens on forms, moving to PDO, and
a lightweight framework. Those are intentionally out of scope here.

## License

[MIT](LICENSE) — free to use, modify, and share. Attribution appreciated.

---

Made to help beginners build real things with Claude Code. Contributions and issues welcome.
