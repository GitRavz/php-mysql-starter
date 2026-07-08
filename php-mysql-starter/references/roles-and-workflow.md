# Roles & Workflow (Multi-User Systems)

> Read this before adding user roles, permission checks, or a status/approval workflow —
> anything where **different people do different things to the same records as those
> records move through stages**. This is the shape of an order-monitoring system (OMS),
> a help desk, an approval tracker. It builds on the single-user CRUD in `boilerplate.md`.

## Table of contents
1. The mental model: roles vs. status
2. Adding a `role` to users
3. Guarding a page by role (`require_role`)
4. The status column — the workflow itself
5. Guarding the *transition*, not just the page
6. Who-can-do-what: a permission table you can read at a glance
7. Logging who did what (activity trail)
8. Worked example — a mini order workflow
9. Golden rules (the traps that get systems hacked)

---

## 1. The mental model: roles vs. status

Two different ideas, easy to confuse:

- **Role** = *what kind of user this is* (sales, manager, production, releasing, admin).
  It lives on the **`users`** row and rarely changes.
- **Status** = *where this record is in its journey* (pending → approved → in_production
  → released). It lives on the **record** (e.g. `orders`) and changes constantly.

A workflow is roles **moving records between statuses**. Keep them separate: never store
"this user is at status X." Users have roles; records have status.

## 2. Adding a `role` to users

Add a `role` column. Two common ways:

```sql
-- Simple (good for a fixed, small set of roles):
ALTER TABLE users
  ADD COLUMN role ENUM('sales','manager','production','releasing','admin')
  NOT NULL DEFAULT 'sales';
```

For a larger or changing set, use a `roles` lookup table + `role_id` FK instead. For a
beginner system with a handful of fixed roles, `ENUM` is fine and simplest.

Store the role in the session **at login** so you don't hit the DB on every page — but
re-read from the DB if a role can change mid-session:

```php
// in login.php, after password_verify() succeeds:
$_SESSION['user_id'] = (int)$user['id'];
$_SESSION['role']    = $user['role'];   // add `role` to the SELECT in login.php
```

## 3. Guarding a page by role (`require_role`)

Extend `includes/auth.php` with a role guard. Call it at the **very top** of a page,
before any HTML — same rule as `require_login()`.

```php
// includes/auth.php  (add below require_login)

function current_role(): ?string {
    return $_SESSION['role'] ?? null;
}

// Allow only the listed roles onto this page. Others get bounced.
function require_role(string ...$allowed): void {
    require_login();                       // must be logged in first
    if (!in_array(current_role(), $allowed, true)) {
        http_response_code(403);
        exit('Forbidden — you do not have access to this page.');
    }
}
```

Usage at the top of a production-only page:

```php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';
require_role('production', 'admin');       // admin can see everything
```

> Why check on the server, not just hide the menu link? Hiding a link stops nobody — a
> user can type the URL directly. The `require_role()` call is the real lock; the hidden
> link is just tidiness.

## 4. The status column — the workflow itself

Give the record a `status` column with a **fixed, known set of values**, and default it
to the first stage:

```sql
ALTER TABLE orders
  ADD COLUMN status ENUM('pending','approved','in_production','released','cancelled')
  NOT NULL DEFAULT 'pending';
```

Define the allowed *forward* moves in ONE place in PHP so the rules live in code you can
read, not scattered across pages:

```php
// The only legal next-steps from each status. Anything not listed is rejected.
const WORKFLOW = [
    'pending'       => ['approved', 'cancelled'],
    'approved'      => ['in_production', 'cancelled'],
    'in_production' => ['released'],
    'released'      => [],            // terminal — nothing after it
    'cancelled'     => [],            // terminal
];
```

## 5. Guarding the *transition*, not just the page

This is the part beginners miss. Two checks before every status change:

1. **Is this move legal in the workflow?** (`in_production` can't jump back to `pending`.)
2. **Is this role allowed to make this move?** (only `production` marks `released`.)

Map roles to the transitions they own, then verify both on the server:

```php
// Which role may set which NEW status:
const ROLE_CAN_SET = [
    'manager'    => ['approved', 'cancelled'],
    'production' => ['in_production', 'released'],
    'sales'      => ['cancelled'],
    'admin'      => ['approved','in_production','released','cancelled'], // override
];

function change_status(mysqli $conn, int $orderId, string $newStatus, string $role): string {
    // read the CURRENT status straight from the DB (never trust the page)
    $stmt = $conn->prepare("SELECT status FROM orders WHERE id = ? LIMIT 1");
    $stmt->bind_param("i", $orderId);
    $stmt->execute();
    $row = $stmt->get_result()->fetch_assoc();
    if (!$row) return 'Order not found.';
    $current = $row['status'];

    // 1) legal workflow move?
    if (!in_array($newStatus, WORKFLOW[$current] ?? [], true)) {
        return "Cannot move an order from '$current' to '$newStatus'.";
    }
    // 2) is this role allowed to set that status?
    if (!in_array($newStatus, ROLE_CAN_SET[$role] ?? [], true)) {
        return "Your role ($role) cannot set status '$newStatus'.";
    }

    $stmt = $conn->prepare("UPDATE orders SET status = ? WHERE id = ? AND status = ?");
    $stmt->bind_param("sis", $newStatus, $orderId, $current); // AND status = current
    $stmt->execute();                                          // avoids double-clicks racing
    return $stmt->affected_rows === 1 ? 'ok' : 'Status changed by someone else — refresh.';
}
```

Two subtleties worth teaching:
- **`WHERE ... AND status = $current`** means if two people click "Approve" at once, only
  the first wins — the second updates 0 rows and gets told to refresh. Cheap safety.
- The current status comes **from the database**, not from a hidden `<input>`. A hidden
  field can be edited in the browser; the DB is the source of truth.

## 6. Who-can-do-what: a permission table you can read at a glance

Keep a plain-English matrix in a comment near `ROLE_CAN_SET` so the rules are obvious to
a human, not just the machine:

```
Role         | View orders | Create | Approve | Start production | Release | Cancel
-------------|-------------|--------|---------|------------------|---------|-------
sales        |  own        |  yes   |   no    |       no         |   no    |  yes
manager      |  all        |  no    |   yes   |       no         |   no    |  yes
production   |  all        |  no    |   no    |       yes        |   yes   |  no
releasing    |  all        |  no    |   no    |       no         |   yes   |  no
admin        |  all        |  yes   |   yes   |       yes        |   yes   |  yes
```

When the table and the code disagree, the code wins — so keep them in sync, and when a
rule changes, change `ROLE_CAN_SET`/`WORKFLOW`, not fifteen separate `if` statements.

## 7. Logging who did what (activity trail)

Monitoring systems live or die on "who moved this, and when?" Add a tiny log table and
write one row on every status change:

```sql
CREATE TABLE activity_log (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id   INT UNSIGNED NOT NULL,
    user_id    INT UNSIGNED NOT NULL,
    action     VARCHAR(100) NOT NULL,      -- e.g. "status: approved -> in_production"
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```php
$stmt = $conn->prepare(
    "INSERT INTO activity_log (order_id, user_id, action) VALUES (?, ?, ?)"
);
$action = "status: $current -> $newStatus";
$stmt->bind_param("iis", $orderId, $userId, $action);
$stmt->execute();
```

If a full log feels like too much at the start, at minimum add `updated_by INT` and
`updated_at DATETIME` columns to the record itself.

## 8. Worked example — a mini order workflow

```sql
CREATE TABLE orders (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    reference   VARCHAR(30) NOT NULL UNIQUE,     -- human-facing code, e.g. "ORD-00042"
    customer    VARCHAR(150) NOT NULL,
    created_by  INT UNSIGNED NOT NULL,           -- FK -> users.id (the sales person)
    status      ENUM('pending','approved','in_production','released','cancelled')
                NOT NULL DEFAULT 'pending',
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX (status), INDEX (created_by),
    CONSTRAINT fk_orders_user FOREIGN KEY (created_by) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

An "advance status" endpoint (POST only, CSRF-checked — see `ui-patterns.md`):

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';
require_login();
check_csrf();                                     // from ui-patterns.md

$orderId   = (int)($_POST['order_id'] ?? 0);
$newStatus = $_POST['new_status'] ?? '';
$result    = change_status($conn, $orderId, $newStatus, current_role());

if ($result === 'ok') {
    // log it, then redirect back with a success flash (Post-Redirect-Get)
    flash('success', "Order updated to $newStatus.");
} else {
    flash('error', $result);
}
header('Location: orders.php');
exit;
```

Notice: role guard + CSRF + workflow check + role-can-set check + activity log, all on
the **server**. The buttons a user sees are cosmetic; these checks are the system.

## 9. Golden rules (the traps that get systems hacked)

1. **Never decide permissions from `$_POST`/`$_GET`.** Read the role from the session (set
   at login), read the current status from the DB. Anything from the browser is a wish,
   not a fact.
2. **Guard the action, not just the view.** Hiding a button is UX; the `require_role()`
   and transition check are the actual security.
3. **Keep the rules in one place** (`WORKFLOW`, `ROLE_CAN_SET`) so they're auditable and
   changeable without hunting through pages.
4. **Use `WHERE status = $current` on the UPDATE** so simultaneous clicks can't
   double-advance a record.
5. **Log status changes** — in a monitoring system, "who moved this?" will be asked, and
   you want an answer, not a shrug.
