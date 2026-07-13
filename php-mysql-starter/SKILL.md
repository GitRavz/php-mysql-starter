---
name: php-mysql-starter
description: Use when building or editing a small PHP + MySQL web app with plain HTML/CSS/JS — scaffolding a project, creating or evolving a database, designing tables or writing SQL, adding login/registration or user roles, building a status/approval workflow or order-monitoring system, adding dashboards, tables, modals, or live-updating lists, or debugging a PHP 500 / blank page. Aimed at beginners driving an AI agent on a PHP project; triggers even when the stack isn't named — "my php project", "connect to the database", "make a login page", "create a table", "add user roles", "track order status", "why is my page blank", "add this to my site".
---

# PHP + MySQL Starter

A starter convention set for small PHP + MySQL web apps built with plain HTML, CSS,
and JavaScript. It exists so that a beginner driving Claude Code gets **correct,
consistent, secure** code on the first try instead of a slightly different shape
every session. When in doubt, follow the pattern here rather than inventing a new one.

**Who this is for:** someone new to using an AI coding assistant on a PHP project.
Prefer clarity over cleverness. Explain *why* a rule matters in one line so the user
learns, don't just assert it.

## Reference files — read the matching one BEFORE the task

- `references/database.md` — read before **creating a database, adding a DB user,
  connecting, backing up/restoring data, or changing a table that already has live
  data** (adding a column safely, renaming one without breaking old code). Covers local
  (phpMyAdmin + MySQL CLI) and shared hosting (cPanel).
- `references/schema-design.md` — read before **designing tables or writing
  `CREATE TABLE`**. Data types, keys, relationships, naming, a full worked example.
- `references/boilerplate.md` — read before **scaffolding a new project or writing
  a connection file, a login/register flow, or a CRUD page**. Copy the code verbatim.
- `references/debugging.md` — read when **a page is blank, throws a 500, or a query
  misbehaves**. A triage checklist, not guesswork.
- `references/roles-and-workflow.md` — read before **adding user roles, permission
  checks, or a status/approval workflow** (e.g. an order moving sales → production →
  releasing, like an OMS). Covers the `role` column, `require_role()`, status columns,
  safe status transitions, **sub-stages within a status (stage chips)**, and a
  **sign-off / approval timeline**.
- `references/ui-patterns.md` — read before **building a dashboard, a data table with
  search/filter/pagination, a modal, a status badge, a "success" message after an
  action, or a list that refreshes itself without a full page reload (live polling)**.
  The reusable UI shapes you'd otherwise rebuild on every page.

## Golden rules (non-negotiable — these are the beginner traps)

1. **Never put user input directly into SQL.** Always use *prepared statements* with
   bound parameters. String-concatenating `$_POST` into a query is how sites get hacked
   (SQL injection). See `boilerplate.md` for the exact pattern.
2. **Never store passwords as plain text.** Hash with `password_hash($pw, PASSWORD_DEFAULT)`
   on register, check with `password_verify()` on login. Never MD5/SHA1, never reversible.
3. **Escape everything you echo back to the page** with `htmlspecialchars()`, so a
   user who types `<script>` can't run it in someone else's browser (XSS).
4. **Keep database credentials in one config file**, and add that file (plus `.env`)
   to `.gitignore` so passwords never land in a public repo. Ship a `config.sample.php`
   with blanks instead.
5. **Guard every protected page at the top** (`require auth`, redirect if not logged in)
   *before* any HTML is printed. A guard placed after output does nothing. When the app
   has roles, also check the role here (`require_role('admin')`) — see
   `references/roles-and-workflow.md`.
6. **Protect every state-changing action with a CSRF token.** Any form or AJAX call that
   inserts/updates/deletes must send a hidden token that the server checks before acting,
   so another site can't trick a logged-in user into firing it. This matters the moment
   more than one person uses the system. Pattern in `references/ui-patterns.md`.

If a request would break rule 1 or 2, don't do it — explain the safe version instead.

## Standard project structure

Scaffold new projects like this (small, flat, predictable). Details + code in
`boilerplate.md`.

```
myapp/
├── config.php            # DB credentials (gitignored)
├── config.sample.php     # committed template with blank values
├── includes/
│   ├── db.php            # ONE mysqli connection, in exception mode
│   ├── auth.php          # session start + require_login() helper
│   ├── header.php        # <head>, CSS links, opening layout
│   └── footer.php        # closing layout, JS
├── assets/
│   ├── css/style.css
│   └── js/app.js
├── index.php             # landing / dashboard
├── login.php  register.php  logout.php
└── (feature pages: e.g. items.php, item_add.php, item_edit.php, item_delete.php)
```

## The database connection (the one canonical way)

Every page includes the same `includes/db.php`. It must turn on **exception mode**:

```php
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);
```

Why this matters: in exception mode a bad query (wrong column, missing table) throws
an error you can *see and fix*. Without it, MySQLi fails silently and you get a blank
page with no clue why. The trade-off: **an unhandled query error becomes an HTTP 500**,
so never reference a column that might not exist, and null-check results before
calling `->fetch_assoc()`. Full file in `boilerplate.md`.

## Talking to the database safely

Read a value from the user, run it through a prepared statement:

```php
$stmt = $conn->prepare("SELECT id, title FROM items WHERE owner_id = ?");
$stmt->bind_param("i", $userId);   // type letter must match the column
$stmt->execute();
$result = $stmt->get_result();
```

Bind type letters: `s` = string/text/date, `i` = integer, `d` = decimal/float.
Binding a date or text as `i` silently stores `0`/`0000-00-00` — match the column type.

## Changing a table that already has live data

On shared hosting you can't always run a migration tool, and a query that names a column
which doesn't exist yet is a **500** (exception mode, above). Two safe habits — full
detail in `references/database.md`:

- **Add missing columns defensively** at the top of the page that needs them, so the app
  heals itself instead of white-screening on an old database:
  ```php
  $chk = $conn->query("SHOW COLUMNS FROM `orders` LIKE 'priority'");
  if ($chk && $chk->num_rows === 0) {
      $conn->query("ALTER TABLE `orders` ADD COLUMN `priority` VARCHAR(20) NULL");
  }
  ```
- **When you rename a column, alias it back** so existing code keeps working:
  `SELECT customer_name AS customer FROM orders`. Old `$row['customer']` still resolves;
  a raw reference to the removed name would 500.

## Building a system with roles and a workflow (the OMS shape)

Many real projects aren't "one user editing their own rows" — they're **several roles
acting on shared records that move through stages**: an order created by *sales*, checked
by a *manager*, worked by *production*, handed over by *releasing*. Order/monitoring
systems, help desks, and approval trackers all share this shape. Full patterns live in
`references/roles-and-workflow.md`; the essentials:

- **One `role` column on `users`** (e.g. `ENUM` or a small `roles` table). Decide
  permissions from the role, never from a hidden form field the user could change.
- **A `status` column on the record** with a fixed set of values (`'pending'`,
  `'in_production'`, `'released'`). The status *is* the workflow.
- **Guard by role at the top of the page** with `require_role(...)`, and **guard the
  transition too** — check on the server that this role is allowed to move this record
  *from its current status to the next one* before you `UPDATE`. A button being hidden in
  the UI is not security; the server check is.
- **Log who changed what, when.** A tiny `activity_log` (or `created_by` / `updated_by`
  columns) turns "who moved this order?" from a mystery into a query.

Do NOT gate features by hiding buttons alone, and do NOT trust a `role` or `status` value
that arrived in `$_POST` — re-read the user's real role from the session/DB every time.

## How to work with Claude Code on a PHP project (teach the user this)

The user gets far better results when they:
- **Paste the real error message**, not "it's broken" — the `error_log` line or the
  on-screen error tells Claude exactly what's wrong (see `debugging.md`).
- **Share the table structure** before asking for a query — paste the `CREATE TABLE`
  or a `DESCRIBE items;` so Claude uses columns that actually exist instead of guessing.
- **Say how they deploy** (uploading files via cPanel/FTP vs. local only). If they
  upload manually, hand them the **complete file to paste**, not a diff, and give the
  **exact path** so they overwrite the right one.
- **Ask for one file at a time** and test it before moving on, rather than a whole app
  in one go — easier to run, easier to debug.

Offer these tips proactively when a beginner seems stuck or is copy-pasting blindly.

## When a page is blank or throws a 500 — diagnose, don't guess

1. Turn on visible errors *locally* (`ini_set('display_errors',1); error_reporting(E_ALL);`
   at the very top) — never leave this on in production.
2. On shared hosting, read the newest lines of the `error_log` file in that folder.
3. Remember exception mode: one bad query kills the whole page. Grep for a column or
   table name that might not exist.
4. State the suspected cause *from the evidence* before editing anything.

Full checklist with the common error messages and their fixes: `references/debugging.md`.
