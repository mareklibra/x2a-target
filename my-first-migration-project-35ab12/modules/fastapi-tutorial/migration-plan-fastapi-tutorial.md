---
source-path: cookbooks/fastapi-tutorial
---

# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook deploys the FastAPI tutorial application from `https://github.com/dibanez/fastapi_tutorial.git` to `/opt/fastapi-tutorial`, installs Python, Git, PostgreSQL, and required development packages, creates a Python virtual environment, installs application dependencies, provisions the PostgreSQL role and database, writes an environment file containing the database connection string, and runs the application through the `fastapi-tutorial` systemd service on port `8000`. PostgreSQL is enabled and started as the supporting database service.

## Service Type and Instances

**Service Type**: Application Server with PostgreSQL database dependency

**Configured Instances**:

- **fastapi-tutorial**: FastAPI application server
  - Location/Path: `/opt/fastapi-tutorial`
  - Repository: `https://github.com/dibanez/fastapi_tutorial.git`
  - Revision: `main`
  - Virtual environment: `/opt/fastapi-tutorial/venv`
  - Port/Socket: `0.0.0.0:8000`
  - Application command: `/opt/fastapi-tutorial/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000`
  - systemd unit: `/etc/systemd/system/fastapi-tutorial.service`
  - systemd service name: `fastapi-tutorial`
  - User: `root`
  - Working directory: `/opt/fastapi-tutorial`
  - Restart policy: `Restart=always`
  - Key configuration: `After=network.target postgresql.service`, virtual environment `bin` directory configured in `PATH`

- **postgresql**: Supporting PostgreSQL database service
  - Location/Path: Distribution-managed PostgreSQL installation
  - Port/Socket: TCP port `5432` by default; Unix socket path is not configured by the cookbook
  - systemd service name: `postgresql`
  - Action: Enabled and started
  - Database user: `fastapi`
  - Database name: `fastapi_db`
  - Database owner: `fastapi`
  - Application connection host: `localhost`
  - Key configuration: Database role receives `fastapi_password`; application privileges are granted on `fastapi_db`

No `.each` loops or collection iterations are present. The cookbook configures one application instance named `fastapi-tutorial` and one supporting database service named `postgresql`.

## File Structure

```text
cookbooks/fastapi-tutorial/recipes/default.rb
```

**Recipes:**
```text
cookbooks/fastapi-tutorial/recipes/default.rb
```

**Providers:**
```text
None
```

**Templates:**
```text
None
```

**Attributes:**
```text
None
```

**Files:**
```text
None
```

The cookbook uses inline Chef resource content for `/opt/fastapi-tutorial/.env` and `/etc/systemd/system/fastapi-tutorial.service`. It does not render ERB templates or deploy static cookbook files using `cookbook_file` or `remote_file`.

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/fastapi-tutorial/recipes/default.rb`):
   - Installs the system packages `python3`, `python3-pip`, `python3-venv`, `git`, `postgresql`, `postgresql-contrib`, and `libpq-dev`.
   - Creates `/opt/fastapi-tutorial` with owner `root`, group `root`, and mode `0755`.
   - Synchronizes `https://github.com/dibanez/fastapi_tutorial.git` at revision `main` into `/opt/fastapi-tutorial`.
   - Expects the repository to provide `/opt/fastapi-tutorial/requirements.txt` and `/opt/fastapi-tutorial/app/main.py`.
   - Creates `/opt/fastapi-tutorial/venv` with `python3 -m venv` when the path does not already exist.
   - Installs dependencies with `/opt/fastapi-tutorial/venv/bin/pip install -r /opt/fastapi-tutorial/requirements.txt` from `/opt/fastapi-tutorial`.
   - Enables and starts the `postgresql` service.
   - Creates PostgreSQL role `fastapi` with password `fastapi_password`.
   - Creates database `fastapi_db` owned by `fastapi`.
   - Grants all privileges on `fastapi_db` to `fastapi`.
   - Creates `/opt/fastapi-tutorial/.env` with owner `root`, group `root`, and mode `0644`, containing:
     ```dotenv
     PROJECT_NAME="FastAPI Tutorial"
     API_VERSION=1.0.0
     DATABASE_URL=postgresql://fastapi:fastapi_password@localhost/fastapi_db
     ```
   - Creates `/etc/systemd/system/fastapi-tutorial.service` with owner `root`, group `root`, and mode `0644`.
   - Configures the systemd unit to run `uvicorn app.main:app` as `root`, use `/opt/fastapi-tutorial` as its working directory, bind to `0.0.0.0:8000`, restart automatically, and start after `network.target` and `postgresql.service`.
   - Defines `execute[systemd_reload]` with `systemctl daemon-reload` and triggers it immediately when the systemd unit changes.
   - Enables and starts the `fastapi-tutorial` service after the repository, virtual environment, dependencies, PostgreSQL service, database objects, environment file, and systemd unit are ready.
   - Iterations: None. There are no `.each` loops.

**Resource summary:**

- `package`: 1 resource containing 7 package names
- `directory`: 1
- `git`: 1
- `execute`: 4
  - `create_venv`
  - `install_dependencies`
  - `create_db_user`
  - `systemd_reload`
- `service`: 2
  - `postgresql`
  - `fastapi-tutorial`
- `file`: 2
  - `/opt/fastapi-tutorial/.env`
  - `/etc/systemd/system/fastapi-tutorial.service`

Total resources: **11**

## Dependencies

**External cookbook dependencies**: None listed in the supplied metadata or execution tree.

**System package dependencies**:

- `python3`
- `python3-pip`
- `python3-venv`
- `git`
- `postgresql`
- `postgresql-contrib`
- `libpq-dev`

**Service dependencies**:

- `postgresql`: Enabled and started by the cookbook; the FastAPI unit declares `After=postgresql.service`.
- `fastapi-tutorial`: Enabled and started by the cookbook after application and database setup.

**Application source dependency**:

- Repository: `https://github.com/dibanez/fastapi_tutorial.git`
- Revision: `main`

**Application file dependencies**:

- `/opt/fastapi-tutorial/requirements.txt`
- `/opt/fastapi-tutorial/app/main.py`

## Credentials

**Detection Summary**: 1 logical database credential detected in 1 source file.

**Source**:
  - **Provider**: Hardcoded in the Chef recipe
  - **URL**: None
  - **Path**: `cookbooks/fastapi-tutorial/recipes/default.rb`

### PostgreSQL FastAPI database password

- **Variable(s)**:
  - No Chef attribute variable is used.
  - Hardcoded password: `fastapi_password`
  - Database user: `fastapi`
  - Connection string: `postgresql://fastapi:fastapi_password@localhost/fastapi_db`
- **Source file(s)**:
  - `cookbooks/fastapi-tutorial/recipes/default.rb`
- **Current storage**: Hardcoded in the recipe and embedded in `/opt/fastapi-tutorial/.env`
- **Usage context**:
  - Creates PostgreSQL role `fastapi`.
  - Authenticates the application to `fastapi_db`.
  - Forms the `DATABASE_URL` value in the application environment file.
- **Migration requirement**:
  - Store the password in Ansible Vault or an approved secret-management system.
  - Use the secret variable in both the PostgreSQL user task and the `.env` file.
  - Use `no_log: true` on credential-handling tasks when appropriate.

No data bags, encrypted data bags, Chef Vault, CyberArk, Conjur, HashiCorp Vault, AWS Secrets Manager, API tokens, TLS certificate paths, private key paths, or additional credential namespaces were detected.

## Checks for the Migration

**Files to verify**:

- `/opt/fastapi-tutorial`
- `/opt/fastapi-tutorial/requirements.txt`
- `/opt/fastapi-tutorial/app/main.py`
- `/opt/fastapi-tutorial/venv`
- `/opt/fastapi-tutorial/.env`
- `/etc/systemd/system/fastapi-tutorial.service`

**Database objects to verify**:

- PostgreSQL role `fastapi`
- PostgreSQL database `fastapi_db`
- Database owner `fastapi`
- `ALL` privileges for `fastapi` on `fastapi_db`

**Service endpoints to check**:

- `fastapi-tutorial`: HTTP endpoint `0.0.0.0:8000`
- `postgresql`: TCP endpoint `localhost:5432` when PostgreSQL uses its default port
- Unix socket: Distribution-default PostgreSQL socket; the cookbook does not define its path

**Templates rendered**:

- None
- `/opt/fastapi-tutorial/.env`: inline file content, rendered once
- `/etc/systemd/system/fastapi-tutorial.service`: inline systemd unit content, rendered once

## Pre-flight checks:

```bash
# Verify required packages on Debian/Ubuntu
dpkg -l python3 python3-pip python3-venv git postgresql postgresql-contrib libpq-dev

# Verify required packages on CentOS/RHEL
rpm -q python3 python3-pip python3-venv git postgresql postgresql-contrib libpq-dev
```

### fastapi-tutorial instance

```bash
# Verify the application directory and ownership
stat -c '%A %U:%G %n' /opt/fastapi-tutorial

# Verify repository state and expected source files
git -C /opt/fastapi-tutorial status --short
git -C /opt/fastapi-tutorial branch --show-current
test -f /opt/fastapi-tutorial/requirements.txt
test -f /opt/fastapi-tutorial/app/main.py

# Verify the virtual environment
test -x /opt/fastapi-tutorial/venv/bin/python
test -x /opt/fastapi-tutorial/venv/bin/pip
test -x /opt/fastapi-tutorial/venv/bin/uvicorn
/opt/fastapi-tutorial/venv/bin/python --version
/opt/fastapi-tutorial/venv/bin/pip --version
/opt/fastapi-tutorial/venv/bin/uvicorn --version

# Verify installed application dependencies
/opt/fastapi-tutorial/venv/bin/pip freeze

# Verify environment file permissions and non-secret values
stat -c '%A %U:%G %n' /opt/fastapi-tutorial/.env
grep -E '^(PROJECT_NAME|API_VERSION)=' /opt/fastapi-tutorial/.env
grep '^DATABASE_URL=' /opt/fastapi-tutorial/.env | sed -E 's#://[^:]+:[^@]+@#://***:***@#'

# Validate the systemd unit
stat -c '%A %U:%G %n' /etc/systemd/system/fastapi-tutorial.service
systemd-analyze verify /etc/systemd/system/fastapi-tutorial.service
systemctl daemon-reload
systemctl cat fastapi-tutorial

# Verify service configuration and state
systemctl show fastapi-tutorial \
  --property=User,WorkingDirectory,ExecStart,Restart,UnitFileState,ActiveState
systemctl status fastapi-tutorial --no-pager
systemctl is-enabled fastapi-tutorial
systemctl is-active fastapi-tutorial

# Verify the process and listening port
ps aux | grep '[u]vicorn app.main:app'
ss -tlnp | grep ':8000'
lsof -iTCP:8000 -sTCP:LISTEN

# Verify HTTP responses and logs
curl -I http://127.0.0.1:8000/
curl -sS http://127.0.0.1:8000/
journalctl -u fastapi-tutorial --no-pager -n 100
```

Expected FastAPI service properties:

- User: `root`
- Working directory: `/opt/fastapi-tutorial`
- Restart policy: `Restart=always`
- Command: `/opt/fastapi-tutorial/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000`

### postgresql instance

```bash
# Verify PostgreSQL service state
systemctl status postgresql --no-pager
systemctl is-enabled postgresql
systemctl is-active postgresql

# Verify local administrative access
sudo -u postgres psql -c "SELECT version();"

# Verify the application role
sudo -u postgres psql -tAc \
  "SELECT rolname FROM pg_roles WHERE rolname = 'fastapi';"

# Verify the application database and owner
sudo -u postgres psql -tAc \
  "SELECT datname, pg_get_userbyid(datdba) FROM pg_database WHERE datname = 'fastapi_db';"
sudo -u postgres psql -d fastapi_db -c "\l+ fastapi_db"

# Verify application connectivity
PGPASSWORD='fastapi_password' psql \
  -h localhost \
  -U fastapi \
  -d fastapi_db \
  -c "SELECT current_user, current_database();"

# Verify the default PostgreSQL TCP port
ss -tlnp | grep ':5432'

# Review PostgreSQL logs
journalctl -u postgresql --since "30 minutes ago" --no-pager
```

Replace the displayed cookbook password with the Ansible Vault value after migration.

### Dependency and recovery checks

```bash
# Verify PostgreSQL ordering configuration for the application
systemctl show fastapi-tutorial --property=After

# Confirm automatic restart configuration
systemctl show fastapi-tutorial --property=Restart

# Restart the application and verify recovery
sudo systemctl restart fastapi-tutorial
systemctl is-active fastapi-tutorial
curl -I http://127.0.0.1:8000/

# Search for application and database failures
journalctl -u fastapi-tutorial --no-pager | grep -Ei 'error|failed|traceback|exception'
journalctl -u fastapi-tutorial --no-pager | grep -Ei 'postgres|database|connection'
systemctl --failed
```