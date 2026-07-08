# Schema Design (Designing Your Tables)

> Read this before designing tables or writing `CREATE TABLE`. Good structure up front
> saves you from painful migrations later. Keep it simple: a small app rarely needs
> more than a handful of tables.

## Table of contents
1. The mental model
2. Choosing column data types
3. Keys: primary keys and foreign keys
4. Naming conventions
5. Timestamps every table should have
6. Worked example — a small "task manager" schema
7. Relationships explained (one-to-many, many-to-many)
8. Gotchas

---

## 1. The mental model

A **table** is a spreadsheet: columns define *what* you store, rows are the actual
records. Design each table around **one kind of thing** — users in `users`, tasks in
`tasks`. Don't cram unrelated data into one giant table; link tables together with IDs
instead (see relationships below).

## 2. Choosing column data types

Pick the smallest type that fits the data. Common ones:

| You're storing | Use | Notes |
|---|---|---|
| Whole number ID / count | `INT` (or `INT UNSIGNED` if never negative) | `BIGINT` only if you'll exceed ~2 billion rows |
| Short text (name, title, email) | `VARCHAR(n)` | pick a sane `n`, e.g. 100–255 |
| Long text (description, notes) | `TEXT` | no length limit needed |
| Money / decimals | `DECIMAL(10,2)` | **never `FLOAT`** for money — rounding errors |
| True/false | `TINYINT(1)` | 0 = false, 1 = true |
| A date only | `DATE` | `YYYY-MM-DD` |
| A date + time | `DATETIME` or `TIMESTAMP` | see timestamps section |
| A fixed set of options | `VARCHAR` + validate in PHP, or `ENUM('a','b')` | ENUM is rigid to change later |

`NULL` means "no value / unknown". Mark a column `NOT NULL` when it must always have a
value (like a title), and allow `NULL` when it's genuinely optional (like a middle name).

## 3. Keys: primary keys and foreign keys

- **Primary key (PK):** every table gets a unique `id`. Standard line:
  `id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY`. The database fills it in automatically
  and guarantees no two rows share an id.
- **Foreign key (FK):** a column that points at another table's id. `tasks.user_id`
  holds the `id` of the `users` row that owns the task. This is how tables connect.
- Add an **index** on columns you filter or join on often (FKs especially) so lookups
  stay fast: `INDEX (user_id)`.

## 4. Naming conventions

Consistency beats personal taste — pick one and never mix:

- Table names: lowercase, plural — `users`, `tasks`, `categories`.
- Column names: lowercase with underscores — `first_name`, `created_at`, `is_done`.
- Foreign keys: `<singular_table>_id` — `user_id`, `category_id`.
- Booleans: prefix with `is_`/`has_` — `is_active`, `has_paid`.

## 5. Timestamps every table should have

Add these to nearly every table — they're cheap and endlessly useful for sorting and
debugging ("when was this created?"):

```sql
created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

## 6. Worked example — a small "task manager" schema

Two tables: users own tasks. This is a complete, copy-pasteable starting point.

```sql
-- USERS: people who log in
CREATE TABLE users (
    id            INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email         VARCHAR(150) NOT NULL UNIQUE,        -- UNIQUE = no duplicate signups
    password_hash VARCHAR(255) NOT NULL,               -- stores password_hash() output
    full_name     VARCHAR(120) NOT NULL,
    is_active     TINYINT(1) NOT NULL DEFAULT 1,
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- TASKS: each belongs to one user
CREATE TABLE tasks (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     INT UNSIGNED NOT NULL,                 -- FK -> users.id
    title       VARCHAR(200) NOT NULL,
    details     TEXT NULL,
    is_done     TINYINT(1) NOT NULL DEFAULT 0,
    due_date    DATE NULL,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX (user_id),
    CONSTRAINT fk_tasks_user
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Notes on the choices above:
- `password_hash VARCHAR(255)` — long enough for any future hashing algorithm.
- `email ... UNIQUE` — the database itself blocks a second signup with the same email.
- `ON DELETE CASCADE` — if a user is deleted, their tasks are auto-deleted too (no orphans).
- `ENGINE=InnoDB` — the modern engine that actually supports foreign keys and transactions.

## 7. Relationships explained

- **One-to-many** (most common): one user has many tasks. Put the FK on the "many"
  side — `tasks.user_id`. That's the example above.
- **Many-to-many:** e.g. tasks can have many tags, and a tag applies to many tasks.
  You need a third **join table**:

```sql
CREATE TABLE task_tags (
    task_id INT UNSIGNED NOT NULL,
    tag_id  INT UNSIGNED NOT NULL,
    PRIMARY KEY (task_id, tag_id),          -- the pair is unique
    FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id)  REFERENCES tags(id)  ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 8. Gotchas

- **Don't store a list in one column** (e.g. `"1,4,9"` of tag ids). It's a pain to
  query and update — use a join table instead.
- **`FLOAT` for money loses cents.** Use `DECIMAL(10,2)`.
- **Design the columns you need now, not 30 "just in case" ones.** Adding a column
  later is a one-line `ALTER TABLE` (see `database.md`); over-designing wastes time.
- **Match your PHP `bind_param` type letters to these columns** — `INT` → `i`,
  `VARCHAR`/`TEXT`/`DATE` → `s`, `DECIMAL` → `d`.
