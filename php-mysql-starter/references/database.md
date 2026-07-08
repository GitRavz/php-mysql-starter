# Database Creation & Management

> Read this before creating a database, adding a DB user, connecting from PHP, or
> backing up / restoring data. Two environments are covered: **local development**
> (your own PC with XAMPP/MAMP/Laragon) and **shared hosting** (cPanel). The SQL is
> the same; only the "where do I type it" differs.

## Table of contents
1. Pick your environment
2. Create a database (local — phpMyAdmin)
3. Create a database (local — MySQL command line)
4. Create a database + user (cPanel / shared hosting)
5. Create a dedicated database user & grant privileges (CLI)
6. Character set — always use utf8mb4
7. Connect from PHP
8. Everyday management tasks
9. Backups & restore (the part beginners forget)
10. Common mistakes

---

## 1. Pick your environment

- **Local development:** install XAMPP, MAMP, or Laragon. You get MySQL/MariaDB +
  phpMyAdmin + Apache in one bundle. Use this to build and test on your own machine.
- **Shared hosting (going live):** most cheap hosts (Namecheap, Hostinger, etc.) give
  you **cPanel**, which has a "MySQL Databases" wizard and phpMyAdmin. Databases and
  users there are usually **prefixed with your cPanel username** automatically
  (e.g. you type `myapp`, the real name becomes `cpaneluser_myapp`).

## 2. Create a database (local — phpMyAdmin)

1. Start XAMPP/MAMP, open `http://localhost/phpmyadmin`.
2. Left sidebar → **New**.
3. Database name: `myapp`. Collation: **`utf8mb4_general_ci`**. Click **Create**.

That's it locally — the default `root` user (often with a blank password) can already
use it. Fine for local dev; **never** use a blank-password root in production.

## 3. Create a database (local — MySQL command line)

Open the terminal (XAMPP → Shell, or your OS terminal if `mysql` is on PATH):

```bash
mysql -u root -p
```

Then:

```sql
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
SHOW DATABASES;      -- confirm it's there
USE myapp;           -- switch into it
```

## 4. Create a database + user (cPanel / shared hosting)

1. cPanel → **MySQL Databases**.
2. **Create New Database** → type `myapp` → Create. Note the *full* name shown
   (e.g. `cpaneluser_myapp`).
3. **MySQL Users → Add New User** → username `myappuser`, a strong password. Note the
   full name (`cpaneluser_myappuser`).
4. **Add User To Database** → pick the user + database → **ALL PRIVILEGES** → Make Changes.
5. Use those **full prefixed names** in your PHP config.

Then open **phpMyAdmin** (also in cPanel) to run `CREATE TABLE` statements, or import
a `.sql` file via the **Import** tab.

## 5. Create a dedicated database user & grant privileges (CLI)

Best practice even locally: don't run your app as `root`. Give it its own user with
rights to only its own database.

```sql
CREATE USER 'myappuser'@'localhost' IDENTIFIED BY 'a-strong-password';
GRANT ALL PRIVILEGES ON myapp.* TO 'myappuser'@'localhost';
FLUSH PRIVILEGES;
```

`myapp.*` means "everything inside the `myapp` database, nothing else" — so if this
user's password leaks, the rest of your server is untouched. That's the whole point.

## 6. Character set — always use utf8mb4

Create databases and tables with `utf8mb4`. Plain `utf8` in MySQL is a 3-byte legacy
encoding that **cannot store emoji or some characters** and will throw errors on them.
`utf8mb4` is the real, full UTF-8. Set it on the database (above), on the table
(`schema-design.md`), and on the PHP connection (`$conn->set_charset('utf8mb4')`).

## 7. Connect from PHP

Put credentials in `config.php` (gitignored). The connection itself lives in
`includes/db.php` — full file in `boilerplate.md`. The essentials:

```php
$config = require __DIR__ . '/../config.php';
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT); // see errors, don't fail silent
$conn = new mysqli($config['host'], $config['user'], $config['pass'], $config['name']);
$conn->set_charset('utf8mb4');
```

- **Local:** host `localhost`, user `root`, blank or your password, name `myapp`.
- **cPanel:** host `localhost` (usually), user `cpaneluser_myappuser`, its password,
  name `cpaneluser_myapp`.

## 8. Everyday management tasks

Run these in phpMyAdmin's **SQL** tab or the `mysql` CLI:

```sql
SHOW TABLES;                          -- list tables in the current DB
DESCRIBE items;                       -- see a table's columns + types (a.k.a. DESC)
SELECT * FROM items LIMIT 20;         -- peek at data (LIMIT so you don't dump millions)
ALTER TABLE items ADD COLUMN notes TEXT NULL;      -- add a column
ALTER TABLE items MODIFY COLUMN title VARCHAR(200) NOT NULL;  -- change a column
ALTER TABLE items DROP COLUMN notes;               -- remove a column
DROP TABLE items;                     -- delete a table AND all its rows (careful!)
```

To rename or empty rather than destroy:

```sql
RENAME TABLE items TO products;
TRUNCATE TABLE items;   -- deletes ALL rows but keeps the table structure
```

⚠️ `DROP` and `TRUNCATE` are irreversible. Back up first (next section).

## 9. Backups & restore (the part beginners forget)

**Export / back up** — do this before any risky change or on a schedule:

- **phpMyAdmin:** select the database → **Export** tab → *Quick* → SQL → Go. You get a
  `.sql` file that recreates everything.
- **Command line (`mysqldump`):** produces the same `.sql` file, scriptable:

```bash
mysqldump -u myappuser -p myapp > myapp_backup_2026-07-08.sql
```

**Restore / import** an existing `.sql` file into a database:

- **phpMyAdmin:** select the (empty) database → **Import** tab → choose file → Go.
- **Command line:**

```bash
mysql -u myappuser -p myapp < myapp_backup_2026-07-08.sql
```

Keep at least one recent backup off the server (download it). A backup that lives only
on the same host you might break isn't really a backup.

## 10. Common mistakes

- **Using the un-prefixed name on cPanel.** The real name is `cpaneluser_myapp`, not
  `myapp`. "Access denied" or "Unknown database" usually means a wrong/short name.
- **Blank-password `root` in production.** Fine locally, dangerous live. Make a real user.
- **Forgetting `FLUSH PRIVILEGES`** after granting rights via CLI.
- **Plain `utf8`** instead of `utf8mb4` — breaks on emoji and some names.
- **No backup before `ALTER`/`DROP`.** Sixty seconds of `mysqldump` saves hours of regret.
