# Odoo 20 — Setup From Scratch (Windows)

A step-by-step guide to set up Odoo 20 on Windows from zero: install the tools, pull **only the `20.0` branch** of Odoo, set up the Python environment and the database, and run the server.

Final folder layout:

```
D:\Mantu\odoo_20\
├── env\              # Python 3.12 virtualenv
├── odoo\             # Odoo 20 core source (branch 20.0)
├── custom_addons\    # your own modules
└── odoo.conf         # server config
```

---

## Step 1 — Install the prerequisites

| Tool | Version | Why |
|---|---|---|
| Git | any recent | to pull the Odoo source |
| Python | **3.12** (3.12–3.14 allowed) | Odoo 20 refuses to start below 3.12 (`MIN_PY_VERSION = (3, 12)` in `odoo/release.py`) |
| PostgreSQL | 14 or newer | Odoo's database |

Install them with `winget` (PowerShell):

```powershell
winget install --id Git.Git -e --source winget
winget install --id Python.Python.3.12 -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id PostgreSQL.PostgreSQL.16 -e --source winget
```

Check them (open a **new** terminal first so PATH is refreshed):

```powershell
git --version
py -0              # the list must contain -V:3.12
psql --version     # if not found, add C:\Program Files\PostgreSQL\16\bin to PATH
```

---

## Step 2 — Create the project folder

```powershell
mkdir D:\Mantu\odoo_20
cd D:\Mantu\odoo_20
mkdir custom_addons
```

---

## Step 3 — Pull only the Odoo 20 branch

The Odoo repository is very large (many versions and years of history). Pull **only the `20.0` branch**:

### Option A — fast, latest code only (recommended)

```powershell
git clone --branch 20.0 --single-branch --depth 1 https://github.com/odoo/odoo.git odoo
```

* `--branch 20.0`: check out the Odoo 20 branch
* `--single-branch`: don't download the other branches (17.0, 18.0, master, …)
* `--depth 1`: only the latest commit, no history (about 1 GB instead of many GB)

### Option B — the 20.0 branch with full history

Use this if you need `git log` / `git blame` on the core code:

```powershell
git clone --branch 20.0 --single-branch https://github.com/odoo/odoo.git odoo
```

### Check the branch

```powershell
cd odoo
git branch          # should show: * 20.0
git log -1 --oneline
cd ..
```

### Get the latest Odoo 20 fixes later

```powershell
cd D:\Mantu\odoo_20\odoo
git pull origin 20.0
```

> If you cloned with `--depth 1` and later want the full history: `git fetch --unshallow`

> **Enterprise (optional):** if you have Odoo Enterprise access, pull the same branch from the private repo:
> `git clone --branch 20.0 --single-branch --depth 1 https://github.com/odoo/enterprise.git enterprise`
> and add `D:\Mantu\odoo_20\enterprise` to `addons_path`.

---

## Step 4 — Create the Python 3.12 virtual environment

Use `py -3.12` explicitly. A plain `python -m venv` may pick an older Python that's earlier on PATH.

```powershell
cd D:\Mantu\odoo_20
py -3.12 -m venv env
env\Scripts\Activate.ps1
python --version    # must print Python 3.12.x
```

> If PowerShell blocks the activate script, run this once:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

---

## Step 5 — Install the Python dependencies

`requirements.txt` is **inside the `odoo` folder**, not in the project root.

```powershell
python -m pip install --upgrade pip setuptools wheel
pip install -r odoo\requirements.txt --timeout 120 --retries 10
```

Windows-only extras that aren't in `requirements.txt`:

```powershell
pip install pywin32 six
```

* **pywin32**: required on Windows (`odoo/tools/osutil.py` imports `win32service`). Without it, Odoo shows the misleading error `Unknown command 'server'`.
* **six**: sometimes missing as a dependency of `python-dateutil`.

> **libsass build error** (`No module named 'distutils.msvc9compiler'`): change the `libsass` lines in `odoo\requirements.txt` to the single line `libsass==0.22.0`, which has a prebuilt Windows wheel, then run the install again.

---

## Step 6 — Create the PostgreSQL user

Odoo doesn't run as the `postgres` superuser. Create a dedicated role that's allowed to create databases:

```powershell
psql -U postgres -h localhost
```

Inside `psql`:

```sql
CREATE ROLE odoo WITH LOGIN CREATEDB PASSWORD 'odoo';
\q
```

Make sure the PostgreSQL service is running: **Services** → `postgresql-x64-16` → Running.

---

## Step 7 — Create `odoo.conf`

Create `D:\Mantu\odoo_20\odoo.conf`:

```ini
[options]
addons_path = D:\Mantu\odoo_20\odoo\addons,D:\Mantu\odoo_20\custom_addons
admin_passwd = 123
db_host = localhost
db_port = 5432
db_user = odoo
db_password = odoo
db_name = odoo20_db
http_port = 9006
proxy_mode = False

limit_request = 104857600
limit_memory_soft = 4294967296
limit_memory_hard = 5368709120
limit_time_cpu = 600
limit_time_real = 1200
limit_time_real_cron = 3600
max_cron_threads = 1
```

* `admin_passwd`: the master password for the database manager. Use a strong one on anything other than your own machine.
* `http_port`: the port Odoo listens on (9006 here, the default is 8069).

---

## Step 8 — Initialize the database (first run only)

A new empty database must be initialized with the `base` module once. Otherwise Odoo fails with `Database odoo20_db not initialized`.

```powershell
cd D:\Mantu\odoo_20
env\Scripts\Activate.ps1
python odoo\odoo-bin -c odoo.conf -d odoo20_db -i base
```

Wait until the log says `HTTP service (werkzeug) running on ...:9006`, then press `Ctrl+C`.

> To start with no demo data, add `--without-demo=all`.

---

## Step 9 — Run Odoo

Every time you start Odoo:

```powershell
cd D:\Mantu\odoo_20
env\Scripts\Activate.ps1
python odoo\odoo-bin -c odoo.conf
```

Open **http://localhost:9006/** and log in with **admin / admin**. Change this password right away.

---

## Step 10 — Custom modules

Put each module in `D:\Mantu\odoo_20\custom_addons\<module_name>\`. It's already on `addons_path`.

```powershell
# install a module
python odoo\odoo-bin -c odoo.conf -d odoo20_db -i <module_name>

# upgrade it after code changes
python odoo\odoo-bin -c odoo.conf -d odoo20_db -u <module_name>
```

Or in the browser: turn on **Developer mode** → **Apps** → **Update Apps List** → search → **Install**.

---

## Troubleshooting

| Error | Cause / fix |
|---|---|
| `AssertionError: Outdated python version` | The venv isn't Python 3.12. Delete `env` and create it again with `py -3.12 -m venv env`. |
| `Unknown command 'server'` | An import is failing silently, usually `pywin32`. To see the real error, run `cd odoo` then `python -c "import odoo.cli.server"`. |
| `No module named 'distutils.msvc9compiler'` | libsass build failure. Pin `libsass==0.22.0` (see Step 5). |
| `No module named 'six'` | `pip install six` |
| `Database ... not initialized` | Run Step 8 (`-i base`). |
| `password authentication failed for user "odoo"` | The role or password in PostgreSQL doesn't match `odoo.conf` (Step 6). |
| `Address already in use` / port busy | Another program is using port 9006. Change `http_port` in `odoo.conf`. |
| `git clone` very slow or fails | Use `--depth 1` (Option A), or run `git config --global http.postBuffer 524288000` and retry. |

---

## Quick reference

```powershell
# one-time setup
git clone --branch 20.0 --single-branch --depth 1 https://github.com/odoo/odoo.git odoo
py -3.12 -m venv env
env\Scripts\Activate.ps1
pip install -r odoo\requirements.txt
pip install pywin32 six
python odoo\odoo-bin -c odoo.conf -d odoo20_db -i base

# daily
env\Scripts\Activate.ps1
python odoo\odoo-bin -c odoo.conf

# update Odoo 20 core
cd odoo; git pull origin 20.0; cd ..
```
