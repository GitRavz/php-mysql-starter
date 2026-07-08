# Debugging (Blank Page / 500 / Broken Queries)

> Read this when a page is blank, throws a 500, or a query misbehaves. Work the
> checklist top to bottom — the answer is almost always in an error message you
> haven't looked at yet. Diagnose from evidence; don't change random lines hoping.

## Table of contents
1. First move: make the error visible
2. Where the error log lives
3. The most common error messages and what they mean
4. A repeatable triage order
5. How to give Claude Code enough to actually fix it

---

## 1. First move: make the error visible

A "white screen of death" means PHP hit a fatal error but errors are hidden.
**Locally**, put this at the very top of the failing page to reveal it:

```php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);
```

⚠️ Remove it (or set `display_errors` to `0`) before going live — you don't want to
show internal errors to visitors. In production, read the log instead (next section).

## 2. Where the error log lives

- **Local (XAMPP):** `xampp/apache/logs/error.log` and `xampp/php/logs/php_error_log`.
- **Shared hosting / cPanel:** an `error_log` file appears **in the same folder as the
  script that failed**. Open it and read the **last few lines** — newest errors are at
  the bottom. cPanel also has a **"Errors"** icon showing recent ones.

Read the actual line. The message names the file and line number of the problem.

## 3. The most common error messages and what they mean

| Message | Cause | Fix |
|---|---|---|
| `Unknown column 'x' in 'field list'` | Query names a column that doesn't exist (typo, or never created) | `DESCRIBE tablename;` to see real columns; fix the name or `ALTER TABLE` to add it |
| `Table 'db.x' doesn't exist` | Wrong table name, or wrong database selected | Check spelling and that `config.php` points at the right DB (prefix on cPanel!) |
| `Access denied for user ...` | Wrong DB username/password, or user lacks rights | Verify credentials; on cPanel add the user to the DB with ALL PRIVILEGES |
| `Call to a member function ... on bool/null` | A query failed and you used the result anyway | Turn on exception mode; null-check before `->fetch_assoc()` |
| `Column count doesn't match value count` | `INSERT` lists N columns but binds a different number of `?` | Match the number of columns, `?` placeholders, and `bind_param` letters |
| `Duplicate entry '...' for key` | Inserting a value that violates a `UNIQUE` column (e.g. email) | Catch code `1062` and show a friendly message (see `boilerplate.md` register) |
| `headers already sent` | Something printed (even a blank line/space) **before** `header()`/`session_start()` | No output before redirects; check for whitespace *before* `<?php` or after `?>` |
| `Undefined array key "x"` | Reading `$_POST['x']` / `$_GET['x']` that isn't set | Use `$_POST['x'] ?? ''` and validate |

## 4. A repeatable triage order

1. **Reproduce** — what exact page/action fails? What did you click?
2. **Reveal the error** — display_errors locally, or read the `error_log`.
3. **Read the message literally** — it names the file, the line, and usually the cause.
4. **Check the obvious three** before anything clever:
   - Does every column/table in the query actually exist? (`DESCRIBE`)
   - Is `config.php` pointing at the right database with valid credentials?
   - Did you null-check the query result before using it?
5. **State the cause, then fix one thing** — re-test before changing anything else.

Remember the exception-mode rule from SKILL.md: **one bad query takes down the whole
page.** So a 500 on a data page is usually a single wrong column or table name.

## 5. How to give Claude Code enough to actually fix it

When asking for help, paste:
- The **exact error text** (from screen or `error_log`), including the file + line.
- The **relevant table structure** (`DESCRIBE tablename;` output or the `CREATE TABLE`).
- The **code around the failing line**, not just "the login is broken".

With those three, the fix is usually one edit. Without them, it's guessing — and
guessing on someone else's database is how new bugs get introduced.
