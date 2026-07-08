# UI Patterns (Dashboards, Tables, Modals, Flash, CSRF)

> Read this before building a dashboard, a data table with search/filter/pagination, a
> modal dialog, a status badge, or a "your changes were saved" message. These are the
> shapes you'd otherwise reinvent on every page — copy them once, reuse everywhere.
> Plain HTML/CSS/JS, no framework. Builds on `boilerplate.md`.

## Table of contents
1. Small helper functions (put these in `includes/helpers.php`)
2. CSRF tokens — protect every state-changing action
3. Flash messages + Post-Redirect-Get (feedback after an action)
4. A reusable data table with search, filter, and pagination
5. Status badges
6. Dashboard summary cards (counts by status)
7. A reusable modal (no framework)

---

## 1. Small helper functions

Repetition you write on every page — collect it once. Create `includes/helpers.php` and
`require` it alongside `db.php`/`auth.php`.

```php
<?php
// Escape for safe HTML output. `e()` is just a short htmlspecialchars.
function e(?string $s): string {
    return htmlspecialchars((string)$s, ENT_QUOTES, 'UTF-8');
}

// Redirect and stop. Always exit after a header redirect.
function redirect(string $to): void {
    header("Location: $to");
    exit;
}
```

Now `<?php echo e($row['title']); ?>` everywhere instead of the long call — same safety,
less noise, more likely a beginner actually does it.

## 2. CSRF tokens — protect every state-changing action

**What it defends against:** another website silently making a logged-in user's browser
POST to yours (delete, approve, transfer). A per-session secret token that only your own
pages know closes that door. Every form that changes data carries it; the server rejects
any POST without a matching one.

```php
// includes/helpers.php  (needs a started session — auth.php does that)

function csrf_token(): string {
    if (empty($_SESSION['csrf'])) {
        $_SESSION['csrf'] = bin2hex(random_bytes(32));
    }
    return $_SESSION['csrf'];
}

// Print this hidden field inside every POST form.
function csrf_field(): string {
    return '<input type="hidden" name="csrf" value="' . e(csrf_token()) . '">';
}

// Call at the top of every page that processes a POST, BEFORE touching the DB.
function check_csrf(): void {
    $sent = $_POST['csrf'] ?? '';
    if (!hash_equals($_SESSION['csrf'] ?? '', $sent)) {
        http_response_code(419);
        exit('Session expired or invalid request. Go back and try again.');
    }
}
```

In a form:

```php
<form method="post" action="order_advance.php">
    <?php echo csrf_field(); ?>
    <input type="hidden" name="order_id" value="<?php echo (int)$order['id']; ?>">
    <button name="new_status" value="approved">Approve</button>
</form>
```

And on the handler:

```php
require_once __DIR__ . '/includes/helpers.php';
check_csrf();     // first line of any POST handler
```

> Use `hash_equals()`, not `==`, to compare tokens — it avoids leaking timing
> information. And only **POST** should change data: never delete/approve via a plain
> `<a href>` GET link, because those can be triggered without a token.

## 3. Flash messages + Post-Redirect-Get

After a POST, **redirect** to a page and show a one-time message there. This stops the
"do you want to resubmit?" browser prompt and the double-insert when someone refreshes.

```php
// includes/helpers.php
function flash(string $type, string $msg): void {   // call before redirect()
    $_SESSION['flash'][] = ['type' => $type, 'msg' => $msg];
}

function render_flash(): string {                    // call once in header.php
    $out = '';
    foreach ($_SESSION['flash'] ?? [] as $f) {
        $out .= '<div class="flash flash-' . e($f['type']) . '">' . e($f['msg']) . '</div>';
    }
    unset($_SESSION['flash']);                        // shown once, then gone
    return $out;
}
```

Flow (Post-Redirect-Get): handler does the work → `flash('success', '...')` →
`redirect('orders.php')` → the list page prints `render_flash()` at the top.

```css
.flash        { padding:.6rem 1rem; border-radius:6px; margin:.5rem 0; }
.flash-success{ background:#e6f4ea; color:#1e7e34; }
.flash-error  { background:#fdecea; color:#c0392b; }
```

## 4. A reusable data table with search, filter, and pagination

Monitoring lists grow to thousands of rows — never `SELECT *` them all. Filter with
prepared statements, page the results, keep the controls in the query string.

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/auth.php';
require_once __DIR__ . '/includes/helpers.php';
require_login();

// --- read controls from the URL, with safe defaults ---
$search = trim($_GET['q'] ?? '');
$status = $_GET['status'] ?? '';                 // '' = all
$page   = max(1, (int)($_GET['page'] ?? 1));
$perPage = 20;
$offset  = ($page - 1) * $perPage;

// --- build WHERE dynamically but safely (values still bound, never concatenated) ---
$where = [];
$types = '';
$args  = [];
if ($search !== '') {
    $where[] = "(reference LIKE ? OR customer LIKE ?)";
    $like = '%' . $search . '%';
    $types .= 'ss'; $args[] = $like; $args[] = $like;
}
if ($status !== '') {
    $where[] = "status = ?";
    $types .= 's'; $args[] = $status;
}
$whereSql = $where ? 'WHERE ' . implode(' AND ', $where) : '';

// --- total count (for page numbers) ---
$countStmt = $conn->prepare("SELECT COUNT(*) AS n FROM orders $whereSql");
if ($types) { $countStmt->bind_param($types, ...$args); }
$countStmt->execute();
$total = (int)$countStmt->get_result()->fetch_assoc()['n'];
$pages = max(1, (int)ceil($total / $perPage));

// --- the page of rows (LIMIT/OFFSET appended; they're ints, cast not bound) ---
$sql = "SELECT id, reference, customer, status, created_at
        FROM orders $whereSql ORDER BY created_at DESC LIMIT ? OFFSET ?";
$types2 = $types . 'ii';
$args2  = array_merge($args, [$perPage, $offset]);
$stmt = $conn->prepare($sql);
$stmt->bind_param($types2, ...$args2);
$stmt->execute();
$rows = $stmt->get_result();

$pageTitle = 'Orders';
require __DIR__ . '/includes/header.php';
?>
<h1>Orders</h1>
<?php echo render_flash(); ?>

<form method="get" class="toolbar">
    <input type="search" name="q" value="<?php echo e($search); ?>" placeholder="Search reference or customer">
    <select name="status">
        <option value="">All statuses</option>
        <?php foreach (['pending','approved','in_production','released','cancelled'] as $s): ?>
            <option value="<?php echo $s; ?>" <?php echo $status === $s ? 'selected' : ''; ?>>
                <?php echo ucwords(str_replace('_',' ',$s)); ?>
            </option>
        <?php endforeach; ?>
    </select>
    <button type="submit">Filter</button>
</form>

<table>
    <thead><tr><th>Ref</th><th>Customer</th><th>Status</th><th>Created</th></tr></thead>
    <tbody>
    <?php while ($o = $rows->fetch_assoc()): ?>
        <tr>
            <td><?php echo e($o['reference']); ?></td>
            <td><?php echo e($o['customer']); ?></td>
            <td><?php echo status_badge($o['status']); ?></td>
            <td><?php echo e($o['created_at']); ?></td>
        </tr>
    <?php endwhile; ?>
    <?php if ($total === 0): ?>
        <tr><td colspan="4">No orders match.</td></tr>
    <?php endif; ?>
    </tbody>
</table>

<?php // pagination — keep existing q/status in the links
$qs = fn($p) => '?' . http_build_query(['q'=>$search,'status'=>$status,'page'=>$p]); ?>
<nav class="pager">
    <?php if ($page > 1): ?><a href="<?php echo $qs($page-1); ?>">&larr; Prev</a><?php endif; ?>
    <span>Page <?php echo $page; ?> of <?php echo $pages; ?> (<?php echo $total; ?> total)</span>
    <?php if ($page < $pages): ?><a href="<?php echo $qs($page+1); ?>">Next &rarr;</a><?php endif; ?>
</nav>
<?php require __DIR__ . '/includes/footer.php'; ?>
```

Key points to teach: search terms are **bound** (`LIKE ?` with a `%..%` argument), never
glued into the SQL; `LIMIT`/`OFFSET` are integers you cast; and the filter/search/page
state travels in the query string so links and refreshes keep it.

## 5. Status badges

One small function turns a raw status into a coloured pill, used in every table:

```php
// includes/helpers.php
function status_badge(string $status): string {
    $label = ucwords(str_replace('_', ' ', $status));
    return '<span class="badge badge-' . e($status) . '">' . e($label) . '</span>';
}
```

```css
.badge { padding:.15rem .5rem; border-radius:999px; font-size:.8rem; font-weight:600; }
.badge-pending       { background:#fff3cd; color:#856404; }
.badge-approved      { background:#d1ecf1; color:#0c5460; }
.badge-in_production { background:#cce5ff; color:#004085; }
.badge-released      { background:#d4edda; color:#155724; }
.badge-cancelled     { background:#f8d7da; color:#721c24; }
```

## 6. Dashboard summary cards (counts by status)

The landing page of a monitoring system is usually "how many are in each stage?" One
grouped query, not one query per status:

```php
$counts = [];
$res = $conn->query("SELECT status, COUNT(*) AS n FROM orders GROUP BY status");
while ($r = $res->fetch_assoc()) { $counts[$r['status']] = (int)$r['n']; }
?>
<div class="cards">
    <?php foreach (['pending','approved','in_production','released'] as $s): ?>
        <a class="card" href="orders.php?status=<?php echo $s; ?>">
            <span class="card-num"><?php echo $counts[$s] ?? 0; ?></span>
            <span class="card-label"><?php echo ucwords(str_replace('_',' ',$s)); ?></span>
        </a>
    <?php endforeach; ?>
</div>
```

Each card links straight into the filtered table above — the dashboard and the list reuse
the same `?status=` filter, so you build the mechanism once.

## 7. A reusable modal (no framework)

A modal is just a hidden overlay you toggle with a class. No library needed.

```html
<!-- the trigger -->
<button type="button" data-modal-open="confirmRelease">Release</button>

<!-- the modal -->
<div class="modal" id="confirmRelease" aria-hidden="true">
  <div class="modal-box" role="dialog" aria-modal="true">
    <h3>Release this order?</h3>
    <p>This marks the order as released and notifies the customer.</p>
    <form method="post" action="order_advance.php">
      <?php echo csrf_field(); ?>
      <input type="hidden" name="order_id" value="<?php echo (int)$order['id']; ?>">
      <div class="modal-actions">
        <button type="button" data-modal-close>Cancel</button>
        <button name="new_status" value="released" class="primary">Yes, release</button>
      </div>
    </form>
  </div>
</div>
```

```css
.modal { position:fixed; inset:0; display:none; align-items:center; justify-content:center;
         background:rgba(0,0,0,.45); }
.modal.open { display:flex; }
.modal-box  { background:#fff; padding:1.25rem; border-radius:10px; max-width:420px; width:90%; }
.modal-actions { display:flex; gap:.5rem; justify-content:flex-end; margin-top:1rem; }
```

```js
// assets/js/app.js  — one handler drives every modal on the page
document.addEventListener('click', (ev) => {
  const opener = ev.target.closest('[data-modal-open]');
  if (opener) document.getElementById(opener.dataset.modalOpen)?.classList.add('open');

  if (ev.target.closest('[data-modal-close]')) ev.target.closest('.modal')?.classList.remove('open');

  // click on the dark backdrop (but not the box) closes it
  if (ev.target.classList.contains('modal')) ev.target.classList.remove('open');
});
document.addEventListener('keydown', (ev) => {
  if (ev.key === 'Escape') document.querySelector('.modal.open')?.classList.remove('open');
});
```

The modal still submits a **normal POST form with a CSRF token** — so even though it looks
fancy, the security is identical to a plain form. The JS only shows/hides; the server
still does every real check.
