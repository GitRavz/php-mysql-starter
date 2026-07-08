# Boilerplate (Copy-Paste Starter Files)

> Read this before scaffolding a project or writing a connection file, a login/register
> flow, or a CRUD page. Copy these verbatim and adjust names — they already follow the
> golden rules in SKILL.md (prepared statements, hashed passwords, escaped output,
> auth guards). Assumes the "task manager" schema from `schema-design.md`.

## Table of contents
1. `config.php` and `config.sample.php`
2. `includes/db.php` — the one connection
3. `includes/auth.php` — sessions + login guard
4. `includes/header.php` / `includes/footer.php`
5. `register.php` — create an account (hashing)
6. `login.php` / `logout.php`
7. A full CRUD page set (list / add / edit / delete)
8. `.gitignore`

---

## 1. `config.php` and `config.sample.php`

`config.php` (real values, **gitignored — never commit this**):

```php
<?php
return [
    'host' => 'localhost',
    'user' => 'myappuser',
    'pass' => 'your-real-password',
    'name' => 'myapp',
];
```

`config.sample.php` (committed so others know what to fill in):

```php
<?php
return [
    'host' => 'localhost',
    'user' => '',
    'pass' => '',
    'name' => '',
];
```

## 2. `includes/db.php` — the one connection

```php
<?php
// Turn on exception mode so query errors are visible, not silent blank pages.
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);

$config = require __DIR__ . '/../config.php';

$conn = new mysqli(
    $config['host'],
    $config['user'],
    $config['pass'],
    $config['name']
);
$conn->set_charset('utf8mb4');
```

## 3. `includes/auth.php` — sessions + login guard

```php
<?php
if (session_status() === PHP_SESSION_NONE) {
    session_start();
}

// Call at the TOP of any page that requires a logged-in user.
function require_login(): void {
    if (empty($_SESSION['user_id'])) {
        header('Location: login.php');
        exit;   // stop the script; a redirect without exit still runs the page
    }
}

function current_user_id(): ?int {
    return $_SESSION['user_id'] ?? null;
}
```

## 4. `includes/header.php` / `includes/footer.php`

`header.php`:

```php
<?php // no auth here; individual pages decide if they need require_login() ?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><?php echo htmlspecialchars($pageTitle ?? 'My App'); ?></title>
    <link rel="stylesheet" href="/assets/css/style.css">
</head>
<body>
<main class="container">
```

`footer.php`:

```php
</main>
<script src="/assets/js/app.js"></script>
</body>
</html>
```

Note the **root-absolute paths** (`/assets/...`) so CSS/JS load correctly no matter how
deep the page folder is.

## 5. `register.php` — create an account (hashing)

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';

$error = '';
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $email = trim($_POST['email'] ?? '');
    $name  = trim($_POST['full_name'] ?? '');
    $pw    = $_POST['password'] ?? '';

    if ($email === '' || $name === '' || strlen($pw) < 8) {
        $error = 'Fill everything in; password must be at least 8 characters.';
    } else {
        // Hash the password — NEVER store the raw one.
        $hash = password_hash($pw, PASSWORD_DEFAULT);

        try {
            $stmt = $conn->prepare(
                "INSERT INTO users (email, password_hash, full_name) VALUES (?, ?, ?)"
            );
            $stmt->bind_param("sss", $email, $hash, $name);
            $stmt->execute();
            header('Location: login.php');
            exit;
        } catch (mysqli_sql_exception $e) {
            // 1062 = duplicate entry (email UNIQUE constraint hit)
            $error = ($e->getCode() === 1062)
                ? 'That email is already registered.'
                : 'Could not register. Please try again.';
        }
    }
}

$pageTitle = 'Register';
require __DIR__ . '/includes/header.php';
?>
<h1>Create account</h1>
<?php if ($error): ?><p class="error"><?php echo htmlspecialchars($error); ?></p><?php endif; ?>
<form method="post" action="register.php">
    <input type="text"     name="full_name" placeholder="Full name" required>
    <input type="email"    name="email"     placeholder="Email" required>
    <input type="password" name="password"  placeholder="Password (8+ chars)" required>
    <button type="submit">Register</button>
</form>
<p>Already have an account? <a href="login.php">Log in</a></p>
<?php require __DIR__ . '/includes/footer.php'; ?>
```

## 6. `login.php` / `logout.php`

`login.php`:

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';

$error = '';
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $email = trim($_POST['email'] ?? '');
    $pw    = $_POST['password'] ?? '';

    $stmt = $conn->prepare(
        "SELECT id, password_hash, is_active FROM users WHERE email = ? LIMIT 1"
    );
    $stmt->bind_param("s", $email);
    $stmt->execute();
    $user = $stmt->get_result()->fetch_assoc();

    // Verify against the HASH. Same generic message whether email or password is wrong,
    // so an attacker can't tell which emails exist.
    if ($user && $user['is_active'] && password_verify($pw, $user['password_hash'])) {
        session_regenerate_id(true);           // prevents session fixation
        $_SESSION['user_id'] = (int)$user['id'];
        header('Location: index.php');
        exit;
    }
    $error = 'Invalid email or password.';
}

$pageTitle = 'Log in';
require __DIR__ . '/includes/header.php';
?>
<h1>Log in</h1>
<?php if ($error): ?><p class="error"><?php echo htmlspecialchars($error); ?></p><?php endif; ?>
<form method="post" action="login.php">
    <input type="email"    name="email"    placeholder="Email" required>
    <input type="password" name="password" placeholder="Password" required>
    <button type="submit">Log in</button>
</form>
<p>No account? <a href="register.php">Register</a></p>
<?php require __DIR__ . '/includes/footer.php'; ?>
```

`logout.php`:

```php
<?php
require_once __DIR__ . '/includes/auth.php';
$_SESSION = [];
session_destroy();
header('Location: login.php');
exit;
```

## 7. A full CRUD page set

CRUD = Create, Read, Update, Delete. Every list/add/edit/delete follows this shape.
Notice: `require_login()` at the top of each, prepared statements everywhere, ownership
checks (`user_id = ?`) so users only touch their own rows, and escaped output.

`items.php` — **list (Read)**:

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';
require_login();
$uid = current_user_id();

$stmt = $conn->prepare(
    "SELECT id, title, is_done, due_date FROM tasks WHERE user_id = ? ORDER BY created_at DESC"
);
$stmt->bind_param("i", $uid);
$stmt->execute();
$tasks = $stmt->get_result();

$pageTitle = 'My Tasks';
require __DIR__ . '/includes/header.php';
?>
<h1>My Tasks</h1>
<a href="item_add.php">+ New task</a>
<table>
    <thead><tr><th>Title</th><th>Due</th><th>Done</th><th>Actions</th></tr></thead>
    <tbody>
    <?php while ($t = $tasks->fetch_assoc()): ?>
        <tr>
            <td><?php echo htmlspecialchars($t['title']); ?></td>
            <td><?php echo htmlspecialchars($t['due_date'] ?? '—'); ?></td>
            <td><?php echo $t['is_done'] ? '✅' : '⬜'; ?></td>
            <td>
                <a href="item_edit.php?id=<?php echo (int)$t['id']; ?>">Edit</a>
                <a href="item_delete.php?id=<?php echo (int)$t['id']; ?>"
                   onclick="return confirm('Delete this task?');">Delete</a>
            </td>
        </tr>
    <?php endwhile; ?>
    </tbody>
</table>
<?php require __DIR__ . '/includes/footer.php'; ?>
```

`item_add.php` — **Create**:

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';
require_login();
$uid = current_user_id();

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $title = trim($_POST['title'] ?? '');
    $due   = $_POST['due_date'] ?: null;   // empty string -> NULL
    if ($title !== '') {
        $stmt = $conn->prepare(
            "INSERT INTO tasks (user_id, title, due_date) VALUES (?, ?, ?)"
        );
        $stmt->bind_param("iss", $uid, $title, $due);
        $stmt->execute();
        header('Location: items.php');
        exit;
    }
}

$pageTitle = 'New Task';
require __DIR__ . '/includes/header.php';
?>
<h1>New Task</h1>
<form method="post" action="item_add.php">
    <input type="text" name="title" placeholder="Task title" required>
    <input type="date" name="due_date">
    <button type="submit">Save</button>
</form>
<?php require __DIR__ . '/includes/footer.php'; ?>
```

`item_edit.php` — **Update** (note the ownership check on both read and write):

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';
require_login();
$uid = current_user_id();
$id  = (int)($_GET['id'] ?? 0);

// Load ONLY if it belongs to this user.
$stmt = $conn->prepare("SELECT * FROM tasks WHERE id = ? AND user_id = ? LIMIT 1");
$stmt->bind_param("ii", $id, $uid);
$stmt->execute();
$task = $stmt->get_result()->fetch_assoc();
if (!$task) { http_response_code(404); exit('Task not found.'); }

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $title  = trim($_POST['title'] ?? '');
    $due    = $_POST['due_date'] ?: null;
    $isDone = isset($_POST['is_done']) ? 1 : 0;
    $stmt = $conn->prepare(
        "UPDATE tasks SET title = ?, due_date = ?, is_done = ? WHERE id = ? AND user_id = ?"
    );
    $stmt->bind_param("ssiii", $title, $due, $isDone, $id, $uid);
    $stmt->execute();
    header('Location: items.php');
    exit;
}

$pageTitle = 'Edit Task';
require __DIR__ . '/includes/header.php';
?>
<h1>Edit Task</h1>
<form method="post" action="item_edit.php?id=<?php echo $id; ?>">
    <input type="text" name="title" value="<?php echo htmlspecialchars($task['title']); ?>" required>
    <input type="date" name="due_date" value="<?php echo htmlspecialchars($task['due_date'] ?? ''); ?>">
    <label><input type="checkbox" name="is_done" <?php echo $task['is_done'] ? 'checked' : ''; ?>> Done</label>
    <button type="submit">Update</button>
</form>
<?php require __DIR__ . '/includes/footer.php'; ?>
```

`item_delete.php` — **Delete** (scoped to the owner):

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';
require_login();
$uid = current_user_id();
$id  = (int)($_GET['id'] ?? 0);

$stmt = $conn->prepare("DELETE FROM tasks WHERE id = ? AND user_id = ?");
$stmt->bind_param("ii", $id, $uid);
$stmt->execute();
header('Location: items.php');
exit;
```

> Note: these delete/update via a link+`confirm()` for beginner simplicity. For a real
> production app, state-changing actions should use a POST form with a CSRF token — a
> good next thing to learn once the basics click.

## 8. `.gitignore`

```
config.php
.env
*.log
/uploads/*
!/uploads/.gitkeep
```

This keeps your real database password and any uploaded files out of the public repo.
