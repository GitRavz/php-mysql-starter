# php-mysql-starter — agent instructions

This folder is a portable coding skill for building small, secure **PHP + MySQL** web
apps with plain HTML/CSS/JS. It works the same in any AI coding agent — Claude Code,
Codex, Cursor, Gemini CLI, GitHub Copilot CLI — because everything is plain Markdown.

`SKILL.md` and this `AGENTS.md` hold the **same guidance**. Agents that load skills read
`SKILL.md`; agents that read an `AGENTS.md` as project/rules context read this file.
Either way, the rules below are non-negotiable and the detailed patterns live in
`references/*.md`.

## When to apply

Apply this skill whenever the task is: scaffolding or editing a PHP/MySQL project;
creating or evolving a database; designing tables or writing SQL; adding
login/registration or user roles; building a status/approval workflow or
order-monitoring system; adding dashboards, tables, modals, or live-updating lists; or
debugging a PHP 500 / blank page. Trigger even when the user doesn't name the stack
("my php project", "make a login page", "add user roles", "why is my page blank").

Prefer clarity over cleverness — the user is likely new to driving an AI agent on PHP.
Explain *why* a rule matters in one line so they learn, and follow the patterns in this
skill instead of inventing a new shape each session.

## Golden rules (never break these)

1. **Never put user input directly into SQL.** Use prepared statements with bound
   parameters. Concatenating `$_POST` into a query = SQL injection.
2. **Never store plain-text passwords.** `password_hash(..., PASSWORD_DEFAULT)` on
   register, `password_verify()` on login. Never MD5/SHA1.
3. **Escape everything echoed to the page** with `htmlspecialchars()` (XSS).
4. **Keep DB credentials in one gitignored config file**; commit a blank
   `config.sample.php`. Never commit real passwords.
5. **Guard every protected page at the top**, before any HTML — `require_login()`, and
   `require_role(...)` when the app has roles. A guard after output does nothing.
6. **Protect every state-changing action with a CSRF token**, and only change data on
   **POST**, never a plain `<a href>` GET link.
7. **Run MySQLi in exception mode** (`mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT)`).
   One consequence: a query naming a missing column becomes an HTTP 500, so guard
   columns that may not exist and null-check results before `->fetch_assoc()`.

If a request would break rule 1 or 2, don't do it — explain the safe version instead.

## Read the matching reference BEFORE the task

- `references/database.md` — create/manage a DB, connect, back up, or **change a table
  with live data** (auto-migrate a missing column, rename one without breaking old code).
- `references/schema-design.md` — designing tables, keys, naming; when a JSON column is
  OK and how to read one whose shape has changed.
- `references/boilerplate.md` — copy-paste `config`, `db.php`, auth, login/register, CRUD.
- `references/roles-and-workflow.md` — roles, `require_role()`, status transitions,
  sub-stages (stage chips), and a sign-off timeline.
- `references/ui-patterns.md` — dashboards, filterable/paginated tables, badges, flash
  messages, modals, and live-updating lists (polling + event delegation).
- `references/debugging.md` — blank page / 500 triage and common error messages.

## Deployment note

Many users deploy by uploading files via cPanel/FTP, not `git push`. When they do, give
the **complete file to paste** (not a diff) and the **exact path**, since files often
share names across folders. Ask for the real error text and the table structure
(`DESCRIBE table;`) before writing a query, so you use columns that actually exist.
