# Odoo 20 — Setup From Scratch (macOS / MacBook)

A step-by-step guide to set up Odoo 20 on a MacBook (Apple Silicon M1–M4 or Intel) from zero: install the tools, pull **only the `20.0` branch** of Odoo, set up the Python environment and the database, and run the server.

Final folder layout:

```
~/odoo_20/
├── env/              # Python 3.12 virtualenv
├── odoo/             # Odoo 20 core source (branch 20.0)
├── custom_addons/    # your own modules
└── odoo.conf         # server config
```

All commands run in the **Terminal** app (zsh).

---

## Step 1 — Install Xcode Command Line Tools

These give you `git` and the C compiler that some Python packages need to build.

```bash
xcode-select --install
```

Click **Install** in the popup and wait for it to finish.

---

## Step 2 — Install Homebrew

Skip this if `brew --version` already works.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

On Apple Silicon, add Homebrew to your PATH. The installer prints these lines at the end:

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

---

## Step 3 — Install Python 3.12, PostgreSQL and the system libraries

Odoo 20 needs **Python 3.12 to 3.14**. It refuses to start on older versions (`MIN_PY_VERSION = (3, 12)` in `odoo/release.py`).

```bash
brew install python@3.12 postgresql@16 git
brew install openssl@3 openldap libmagic pkg-config
```

Some of these are needed to build Odoo's Python packages:

| Library | Needed by |
|---|---|
| `postgresql@16` (provides `pg_config`) | `psycopg2` |
| `openssl@3` | `psycopg2`, `cryptography` |
| `openldap` | `python-ldap` |
| `libmagic` | `python-magic` |

Put the PostgreSQL tools on your PATH:

```bash
echo 'export PATH="$(brew --prefix postgresql@16)/bin:$PATH"' >> ~/.zprofile
source ~/.zprofile
```

Check everything:

```bash
python3.12 --version   # Python 3.12.x
psql --version         # psql (PostgreSQL) 16.x
pg_config --version
git --version
```

---

## Step 4 — Start PostgreSQL and create the Odoo user

Start PostgreSQL. It will also start automatically at login from now on:

```bash
brew services start postgresql@16
```

With Homebrew, the PostgreSQL superuser is **your Mac username**, not `postgres`. Create a dedicated `odoo` role that's allowed to create databases:

```bash
psql postgres
```

Inside `psql`:

```sql
CREATE ROLE odoo WITH LOGIN CREATEDB PASSWORD 'odoo';
\q
```

---

## Step 5 — Create the project folder

```bash
mkdir -p ~/odoo_20/custom_addons
cd ~/odoo_20
```

---

## Step 6 — Pull only the Odoo 20 branch

The Odoo repository is very large (many versions and years of history). Pull **only the `20.0` branch**:

### Option A — fast, latest code only (recommended)

```bash
git clone --branch 20.0 --single-branch --depth 1 https://github.com/odoo/odoo.git odoo
```

* `--branch 20.0`: check out the Odoo 20 branch
* `--single-branch`: don't download the other branches (17.0, 18.0, master, …)
* `--depth 1`: only the latest commit, no history (about 1 GB instead of many GB)

### Option B — the 20.0 branch with full history

Use this if you need `git log` / `git blame` on the core code:

```bash
git clone --branch 20.0 --single-branch https://github.com/odoo/odoo.git odoo
```

### Check the branch

```bash
cd odoo
git branch          # should show: * 20.0
git log -1 --oneline
cd ..
```

### Get the latest Odoo 20 fixes later

```bash
cd ~/odoo_20/odoo
git pull origin 20.0
```

> If you cloned with `--depth 1` and later want the full history: `git fetch --unshallow`

> **Enterprise (optional):** if you have Odoo Enterprise access, pull the same branch from the private repo:
> `git clone --branch 20.0 --single-branch --depth 1 https://github.com/odoo/enterprise.git enterprise`
> and add `~/odoo_20/enterprise` (as a full path) to `addons_path`.

---

## Step 7 — Create the Python 3.12 virtual environment

Use `python3.12` explicitly. macOS also ships its own older `python3`, which is too old for Odoo 20.

```bash
cd ~/odoo_20
python3.12 -m venv env
source env/bin/activate
python --version    # must print Python 3.12.x
```

---

## Step 8 — Install the Python dependencies

`requirements.txt` is **inside the `odoo` folder**, not in the project root.

First, tell the compiler where the Homebrew libraries are. Without these lines, the `psycopg2` and `python-ldap` builds fail:

```bash
export LDFLAGS="-L$(brew --prefix openssl@3)/lib -L$(brew --prefix openldap)/lib"
export CPPFLAGS="-I$(brew --prefix openssl@3)/include -I$(brew --prefix openldap)/include -I$(xcrun --show-sdk-path)/usr/include/sasl"
```

Then install:

```bash
python -m pip install --upgrade pip setuptools wheel
pip install -r odoo/requirements.txt --timeout 120 --retries 10
```

> You don't need `pywin32` on a Mac. It's Windows-only.

---

## Step 9 — Create `odoo.conf`

Create `~/odoo_20/odoo.conf`. Replace `YOUR_USERNAME` with the output of `whoami`, because `addons_path` needs full paths:

```ini
[options]
addons_path = /Users/YOUR_USERNAME/odoo_20/odoo/addons,/Users/YOUR_USERNAME/odoo_20/custom_addons
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

## Step 10 — Initialize the database (first run only)

A new empty database must be initialized with the `base` module once. Otherwise Odoo fails with `Database odoo20_db not initialized`.

```bash
cd ~/odoo_20
source env/bin/activate
python odoo/odoo-bin -c odoo.conf -d odoo20_db -i base
```

Wait until the log says `HTTP service (werkzeug) running on ...:9006`, then press `Ctrl+C`.

> To start with no demo data, add `--without-demo=all`.

---

## Step 11 — Run Odoo

Every time you start Odoo:

```bash
cd ~/odoo_20
source env/bin/activate
python odoo/odoo-bin -c odoo.conf
```

Open **http://localhost:9006/** and log in with **admin / admin**. Change this password right away.

---

## Step 12 — Custom modules

Put each module in `~/odoo_20/custom_addons/<module_name>/`. It's already on `addons_path`.

```bash
# install a module
python odoo/odoo-bin -c odoo.conf -d odoo20_db -i <module_name>

# upgrade it after code changes
python odoo/odoo-bin -c odoo.conf -d odoo20_db -u <module_name>
```

Or in the browser: turn on **Developer mode** → **Apps** → **Update Apps List** → search → **Install**.

---

## Troubleshooting

| Error | Cause / fix |
|---|---|
| `AssertionError: Outdated python version` | The venv isn't Python 3.12. Run `rm -rf env`, then `python3.12 -m venv env`. |
| `Error: pg_config executable not found` (psycopg2) | PostgreSQL isn't on PATH. Redo the PATH step in Step 3 and open a new terminal. |
| `ld: library not found for -lssl` (psycopg2) | The `LDFLAGS` export from Step 8 is missing. Run it in the same terminal, then install again. |
| `fatal error: 'sasl.h' file not found` (python-ldap) | The `CPPFLAGS` export from Step 8 is missing. Run it, then install again. |
| `failed to find libmagic` | `brew install libmagic` |
| `connection to server ... failed: Connection refused` | PostgreSQL isn't running. Run `brew services start postgresql@16`. |
| `role "postgres" does not exist` | Normal with Homebrew. Connect with `psql postgres`, which uses your Mac username. |
| `password authentication failed for user "odoo"` | The role or password doesn't match `odoo.conf` (Step 4). |
| `Address already in use` / port busy | Another program is using port 9006. Change `http_port`, or find it with `lsof -i :9006`. |
| `git clone` very slow or fails | Use `--depth 1` (Option A), or run `git config --global http.postBuffer 524288000` and retry. |

---

## Quick reference

```bash
# one-time setup
xcode-select --install
brew install python@3.12 postgresql@16 git openssl@3 openldap libmagic pkg-config
brew services start postgresql@16
psql postgres -c "CREATE ROLE odoo WITH LOGIN CREATEDB PASSWORD 'odoo';"

mkdir -p ~/odoo_20/custom_addons && cd ~/odoo_20
git clone --branch 20.0 --single-branch --depth 1 https://github.com/odoo/odoo.git odoo
python3.12 -m venv env && source env/bin/activate
export LDFLAGS="-L$(brew --prefix openssl@3)/lib -L$(brew --prefix openldap)/lib"
export CPPFLAGS="-I$(brew --prefix openssl@3)/include -I$(brew --prefix openldap)/include -I$(xcrun --show-sdk-path)/usr/include/sasl"
pip install -r odoo/requirements.txt
python odoo/odoo-bin -c odoo.conf -d odoo20_db -i base

# daily
cd ~/odoo_20 && source env/bin/activate
python odoo/odoo-bin -c odoo.conf

# update Odoo 20 core
cd ~/odoo_20/odoo && git pull origin 20.0
```
