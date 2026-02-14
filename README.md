# MyBusiness Admin Portal — Full Project Summary

## Overview

A Laravel 11 admin portal deployed at **office.mybusiness.com.my** on shared hosting (cPanel). Migrated from an older Laravel project, this system provides a full back-office management platform with user management, role-based permissions, dynamic menus, file management, backup system, and database manager.

---

## Hosting & Environment

| Item | Detail |
|---|---|
| **URL** | `https://office.mybusiness.com.my` |
| **Server Path** | `/home/mybusiness/office.mybusiness.com.my` |
| **Hosting** | Shared hosting (cPanel), no SSH, no queue workers |
| **PHP** | 8.2+ |
| **Database** | MariaDB (MySQL-compatible) |
| **DB Name** | `mybusiness_b2b` |
| **Framework** | Laravel 11 |
| **Login** | Username: `roderisland` / Password: `zhubajie123!@#` |

### Shared Hosting Constraints
- No `exec()`, `escapeshellarg()`, `shell_exec()` — all disabled
- No queue workers — backups run synchronously via AJAX
- No SSH access — all deployments via cPanel File Manager
- Public folder served from project root (`.htaccess` rewrites to `public/index.php`)
- `highlight_file()` disabled — Symfony error pages may break (non-critical)

### Second Site
There's also `apps.mybusiness.com.my` at `/home/mybusiness/apps.mybusiness.com.my` — a separate older project that gets included in full-project backups.

---

## Authentication System

**Cookie-based auth** (not Laravel's built-in session guard). This was carried over from the old portal.

### How It Works
1. User submits username + password to `LoginController@login`
2. System finds user in `tbl_admin` by username or email
3. Verifies password with `Hash::check()`
4. Sets an `admin_id` cookie (HTTP-only, 7-day expiry)
5. All subsequent requests read `admin_id` from cookie
6. `AdminAuthenticate` middleware checks cookie on every request

### Key Files
- **Controller**: `app/Http/Controllers/Admin/Auth/LoginController.php`
- **Middleware**: `app/Http/Middleware/AdminAuthenticate.php` — checks `admin_id` cookie
- **Middleware**: `app/Http/Middleware/RedirectIfAdminAuthenticated.php` — redirects logged-in users away from login page
- **Middleware**: `app/Http/Middleware/CheckAdminMenuAccess.php` — checks role-based menu permissions
- **Model**: `app/Models/Admin.php` — extends `Authenticatable`, table `tbl_admin`

### Getting Current Admin in Code
```php
$adminId = request()->cookie('admin_id');
$admin = \App\Models\Admin::find($adminId);
```

---

## Middleware Registration

Defined in `bootstrap/app.php` (Laravel 11 style):

```php
$middleware->alias([
    'admin.auth'  => \App\Http\Middleware\AdminAuthenticate::class,
    'admin.guest' => \App\Http\Middleware\RedirectIfAdminAuthenticated::class,
    'admin.role'  => \App\Http\Middleware\CheckAdminRole::class,
    'admin.access'=> \App\Http\Middleware\CheckAdminMenuAccess::class,
]);
```

### Route Groups
- **Guest routes** (login): `Route::middleware('admin.guest')`
- **Authenticated routes**: `Route::middleware(['admin.auth', 'admin.access'])`

---

## Routing

All admin routes are in `routes/admin.php`, loaded in `routes/web.php` via:
```php
Route::group([], base_path('routes/admin.php'));
```

The root `/` redirects to `admin.login`. There is **no `/admin` prefix** — all routes are at the root level (e.g., `/users`, `/backup`, `/database`).

---

## Database Schema

### Core Admin Tables

| Table | Purpose |
|---|---|
| `tbl_admin` | Admin users (id, name, email, username, password, role_id, is_active, datetime_lastlogin) |
| `tbl_admin_roles` | Roles (id, name, slug, description, is_active). Default roles: administrator, supervisor, staff |
| `tbl_admin_menu_groups` | Menu groups/sections (id, name, slug, sort_order, is_active) |
| `tbl_admin_menus` | Menu items (id, group_id, parent_id, level, title, icon, route_name, url, permission_key, sort_order, is_active) |
| `tbl_admin_role_menu_access` | Permission matrix (role_id, menu_id, can_view, can_create, can_edit, can_delete) |

### Business Tables (migrated from old portal)

| Table | Purpose |
|---|---|
| `tbl_company` | Companies |
| `tbl_company_admin` | Company admin users |
| `tbl_company_admin_access_log` | Company admin login logs |
| `tbl_company_admin_email_log` | Company admin email logs |
| `tbl_company_admin_role` | Company admin roles |
| `tbl_partner` | Partners |
| `tbl_partner_access_log` | Partner login logs |
| `tbl_partner_bankdetails` | Partner bank details |
| `tbl_partner_email_log` | Partner email logs |
| `tbl_marketing_campaign` | Marketing campaigns |
| `tbl_marketing_campaign_data` | Campaign data records |
| `tbl_marketing_campaign_partner` | Campaign-partner links |
| `tbl_marketing_data` | Marketing data |
| `tbl_system_bank` | System bank list |
| `tbl_system_email_smtp` | SMTP settings |
| `tbl_system_email_templates` | Email templates |

### Backup Tables

| Table | Purpose |
|---|---|
| `tbl_backup_jobs` | Backup job definitions (name, frequency, cron, include/exclude paths, exclude_extensions, retention) |
| `tbl_backup_runs` | Backup execution records (job_id, folder_name, status, progress, size, timestamps) |
| `tbl_backup_logs` | Per-run log entries (run_id, level, message, file_path, file_size, logged_at) |

### Menu Items in Database (tbl_admin_menus)

| ID | Group | Title | Route | Icon |
|---|---|---|---|---|
| 1 | Main Menu | Dashboard | admin.dashboard | fas fa-tachometer-alt |
| 2 | Management | Admin Users | admin.users.index | fas fa-users |
| 3 | Management | Roles | admin.roles.index | fas fa-user-shield |
| 4 | Management | Menu Management | admin.menus.index | fas fa-bars |
| 5 | Management | Permissions | admin.permissions.index | fas fa-key |
| 6 | Management | File Manager | admin.filemanager.index | fas fa-folder-open |
| 7 | Reports | Sales Report | admin.reports.sales | fas fa-chart-bar |
| 8 | Reports | Analytics | admin.reports.analytics | fas fa-chart-line |
| 9 | Settings | General | admin.settings.general | fas fa-cog |
| 10 | Settings | Security | admin.settings.security | fas fa-shield-alt |
| 14 | Management | Backup | admin.backup.index | fas fa-database |
| 15 | Management | Database | admin.database.index | fas fa-server |

---

## File Structure

```
├── app/
│   ├── Console/Commands/
│   │   ├── BackupRunCommand.php          # php artisan backup:run {job?}
│   │   └── BackupRestoreCommand.php      # php artisan backup:restore {id}
│   ├── Http/
│   │   ├── Controllers/Admin/
│   │   │   ├── Auth/LoginController.php  # Login/logout (cookie-based)
│   │   │   ├── AdminController.php       # Admin user CRUD
│   │   │   ├── BackupController.php      # Backup module (16 methods)
│   │   │   ├── DashboardController.php   # Dashboard
│   │   │   ├── DatabaseController.php    # Database manager (phpMyAdmin)
│   │   │   ├── FileManagerController.php # File manager wrapper
│   │   │   ├── MenuController.php        # Menu & permission management
│   │   │   └── RoleController.php        # Role CRUD
│   │   └── Middleware/
│   │       ├── AdminAuthenticate.php         # Cookie auth check
│   │       ├── CheckAdminMenuAccess.php      # Role permission check
│   │       ├── CheckAdminRole.php            # Role check
│   │       └── RedirectIfAdminAuthenticated.php  # Guest redirect
│   ├── Jobs/
│   │   ├── RunBackupJob.php              # Async backup (not used on shared hosting)
│   │   └── RestoreBackupJob.php          # Async restore (not used on shared hosting)
│   ├── Models/
│   │   ├── Admin.php                     # tbl_admin
│   │   ├── AdminMenu.php                # tbl_admin_menus
│   │   ├── AdminMenuGroup.php           # tbl_admin_menu_groups
│   │   ├── AdminRole.php                # tbl_admin_roles
│   │   ├── AdminRoleMenuAccess.php      # tbl_admin_role_menu_access
│   │   ├── BackupJob.php                # tbl_backup_jobs
│   │   ├── BackupLog.php                # tbl_backup_logs
│   │   ├── BackupRun.php                # tbl_backup_runs
│   │   └── User.php                     # Laravel default (unused)
│   ├── Providers/AppServiceProvider.php
│   └── Services/
│       └── BackupService.php             # Core backup engine (execute, restore, collectFiles, phpDatabaseDump)
├── bootstrap/app.php                     # Middleware aliases registered here
├── config/
│   ├── auth.php                          # Default web guard only (admin uses cookies)
│   ├── file-manager.php                  # File manager disk config
│   └── filesystems.php                   # 'home' disk pointing to /home/mybusiness
├── resources/views/admin/
│   ├── auth/login.blade.php
│   ├── layouts/app.blade.php             # Main layout (left sidebar, top bar, content area)
│   ├── partials/
│   │   ├── menu_left.blade.php           # Dynamic sidebar from DB
│   │   ├── menu_upper.blade.php          # Top navigation bar
│   │   └── menu_footer.blade.php
│   └── pages/
│       ├── dashboard.blade.php
│       ├── backup/                       # 5 views: index, jobs, history, logs, restore
│       ├── database/                     # 5 views: index, table, query, export, import
│       ├── filemanager/index.blade.php
│       ├── menus/index.blade.php
│       ├── permissions/index.blade.php
│       ├── roles/index.blade.php, edit.blade.php
│       ├── users/index.blade.php, edit.blade.php
│       ├── reports/sales.blade.php, analytics.blade.php    # Placeholder pages
│       └── settings/general.blade.php, security.blade.php  # Placeholder pages
├── routes/
│   ├── web.php                           # Root redirect + vendor-asset route + file-manager route
│   └── admin.php                         # All admin routes (no /admin prefix)
└── migration.sql                         # Full database schema + seed data
```

---

## Modules

### 1. Admin Users (`/users`)
- List all admin users with role, status, last login
- Create/edit users with name, email, username, password, role
- Toggle active/inactive
- **Controller**: `AdminController.php`

### 2. Roles (`/roles`)
- CRUD for roles (administrator, supervisor, staff, custom)
- Each role has slug, description, active status
- **Controller**: `RoleController.php`

### 3. Menu Management (`/menus`)
- Dynamic menu system stored in database
- Menu groups (sections) and menu items
- Drag-and-drop reordering via AJAX
- Parent-child menu hierarchy (level 1 and 2)
- Each item: title, icon (Font Awesome), route_name, permission_key
- **Controller**: `MenuController.php`

### 4. Permissions (`/permissions`)
- Matrix view: roles × menu items
- Each cell has: can_view, can_create, can_edit, can_delete
- Administrator role has full access by default (hardcoded in `Admin` model)
- **Controller**: `MenuController@permissions`

### 5. File Manager (`/filemanager`)
- Built on `alexusmai/laravel-file-manager` package
- Disk: `home` → `/home/mybusiness` (full server access)
- Vendor assets served via custom Laravel route (`/vendor-asset/{path}`) because shared hosting can't symlink to public
- File editor with Monaco editor integration
- **Controller**: `FileManagerController.php`
- **Config**: `config/file-manager.php`, `config/filesystems.php`

### 6. Backup System (`/backup`)
Full-featured backup and restore system.

#### Pages
- **Dashboard** (`/backup`): Stats cards, quick actions, recent runs with progress bars
- **Jobs** (`/backup/jobs`): Create/edit backup jobs with frequency, include/exclude paths, exclude extensions, database toggle, retention count
- **History** (`/backup/history`): Paginated list of all backup runs with status, progress, size
- **Logs** (`/backup/logs/{id}`): Live terminal-style log viewer with real-time AJAX updates during backup, pagination for completed backups
- **Restore** (`/backup/restore/{id}`): Restore wizard with confirmation

#### How Backups Work
1. User clicks "Run" on a job → `BackupController@runNow` creates a `BackupRun` record with status `pending` and redirects to logs page
2. Logs page detects `pending` status → fires AJAX POST to `/backup/execute/{id}`
3. `BackupService@execute()` runs synchronously in that AJAX request (no queue on shared hosting)
4. Every 1.5 seconds, logs page polls `/backup/progress/{id}?after_id=X` for new logs and progress
5. Logs stream in live — progress bar, file count, size, duration timer all update in real-time
6. On completion, page auto-reloads to show final state with pagination

#### Backup Storage
```
/backup/backup_2026-02-14_015219/
├── files/          # Copied files preserving directory structure
├── database.sql    # PHP-based MySQL dump (no shell commands)
└── metadata.json   # Backup details (job, files, size, paths, versions)
```

#### Key Technical Details
- **File collection**: Supports both absolute paths (`/home/mybusiness/office.mybusiness.com.my`) and relative paths (`app`, `config`)
- **Extension exclusion**: Configurable per job (e.g., zip, tar, gz, mp4, log, tmp)
- **Default excludes**: vendor, node_modules, backup, .git, storage/logs, storage/framework/cache
- **Database dump**: Pure PHP — reads all tables via PDO, generates INSERT statements. No `mysqldump` or shell commands
- **Progress logging**: Logs per-directory summaries + milestone every 5% to keep log size manageable
- **DB updates**: Every 20 files for responsive progress bar
- **Retention**: Auto-deletes old backups beyond `retention_count` after each successful backup
- **Restore**: Copies files back to original locations, imports SQL dump statement-by-statement

#### Controller Methods (BackupController.php)
- `index()` — Dashboard with stats
- `jobs()` / `storeJob()` / `updateJob()` / `deleteJob()` / `toggleJob()` — Job CRUD
- `runNow($jobId)` — Create run + redirect to logs
- `runManual()` — One-off backup
- `executeRun($runId)` — AJAX: actually runs the backup
- `history()` — Paginated run history
- `logs($runId)` — Log viewer (auto-redirects to last page for completed backups)
- `progress($runId)` — AJAX: returns JSON with status, progress, new logs after given ID
- `restoreConfirm($runId)` — Restore wizard
- `restoreExecute($runId)` — Mark as restoring + redirect
- `restoreExecuteAjax($runId)` — AJAX: actually runs restore
- `deleteBackup($runId)` — Delete backup files + DB records
- `deleteLogs($runId)` — Clear log entries for a run

### 7. Database Manager (`/database`)
A phpMyAdmin replacement built into the admin portal.

#### Pages
- **Tables** (`/database`): List all tables with engine, rows, size, collation. Search filter. Truncate/drop actions
- **Table View** (`/database/table/{name}`): Tabs for Data (paginated, 50/page), Structure (columns, types, keys), Indexes, SQL (CREATE TABLE). Delete individual rows
- **SQL Query** (`/database/query`): Dark-themed SQL editor, Ctrl+Enter execution, quick query buttons (SHOW TABLES, VERSION, etc.), result table with execution time, blocked dangerous operations (DROP DATABASE, GRANT, etc.)
- **Export** (`/database/export`): Select tables, choose structure/data, downloads .sql file
- **Import** (`/database/import`): Upload .sql file (max 50MB), auto-executes all statements with proper string-aware splitting

#### Controller Methods (DatabaseController.php)
- `index()` — Table list with SHOW TABLE STATUS
- `viewTable($table)` — Structure + paginated data + indexes + CREATE SQL
- `query()` — Execute arbitrary SQL (GET for form, POST for execution)
- `export()` — GET for form, POST generates and downloads .sql
- `import()` — GET for form, POST processes uploaded .sql file
- `dropTable($table)` — DROP TABLE
- `truncateTable($table)` — TRUNCATE TABLE
- `deleteRow($table)` — DELETE single row by primary key

---

## UI / Design

- **CSS Framework**: Custom CSS (no Bootstrap/Tailwind). Clean admin style
- **Font**: Inter (Google Fonts CDN)
- **Primary Color**: `#4f46e5` (Indigo)
- **Icons**: Font Awesome 5 (CDN)
- **Layout**: Fixed left sidebar + top bar + scrollable content area
- **Sidebar**: Dynamic from `tbl_admin_menus` → grouped by `tbl_admin_menu_groups`
- **Components**: Card-based, status badges, progress bars, modals, toast notifications
- **Terminal**: Dark theme (`#0f172a`) monospace log viewer for backup logs

---

## Key Configuration Files

### `config/filesystems.php`
```php
'home' => [
    'driver' => 'local',
    'root' => '/home/mybusiness',
],
```

### `config/file-manager.php`
```php
'diskList' => ['home'],
'leftDisk' => 'home',
'rightDisk' => null,
```

### `bootstrap/app.php`
All middleware aliases registered here. Routes loaded from `routes/web.php` → includes `routes/admin.php`.

---

## Common Issues & Solutions

| Issue | Cause | Solution |
|---|---|---|
| `escapeshellarg()` undefined | Shared hosting disables shell functions | All DB dumps use pure PHP, no shell commands |
| Backup stuck at "Pending" | Queue dispatch but no worker | Backup runs synchronously via AJAX call |
| `SHOW TABLE STATUS LIKE ?` error | MariaDB doesn't accept `?` placeholder for LIKE | Use direct string interpolation with validated table name |
| Vendor assets 404 | Can't symlink on shared hosting | Custom `/vendor-asset/{path}` route serves files directly |
| Duration shows negative | `diffInSeconds()` order matters | Wrapped in `abs()` |
| 0 files found in backup | Full absolute paths appended to base_path | `collectFiles()` detects absolute paths and uses them directly |
| URL has `/public` or `/admin` prefix | Incorrect routing/deployment | `.htaccess` in root rewrites to `public/`, routes have no prefix |

---

## Deployment Process

Since there's no SSH, deployment is manual via cPanel File Manager:

1. Upload updated files via cPanel File Manager (or zip upload + extract)
2. For database changes, run SQL via the Database Manager module (`/database/query`)
3. Clear Laravel cache by deleting files in `storage/framework/cache/` and `bootstrap/cache/`

### SQL for Adding New Menu Items
```sql
INSERT INTO tbl_admin_menus (id, group_id, parent_id, level, title, icon, route_name, url, permission_key, sort_order, is_active, created_at, updated_at)
VALUES (15, 3, NULL, 1, 'Database', 'fas fa-server', 'admin.database.index', NULL, 'database', 5, 1, NOW(), NOW());
```

---

## What's NOT Built Yet (placeholder pages)

These routes exist with basic placeholder views:
- Reports > Sales Report (`/reports/sales`)
- Reports > Analytics (`/reports/analytics`)
- Settings > General (`/settings/general`)
- Settings > Security (`/settings/security`)
- Activity Log (`/activity-log`)

The business tables (tbl_company, tbl_partner, tbl_marketing_*) are migrated in the database but have no UI modules built yet.
