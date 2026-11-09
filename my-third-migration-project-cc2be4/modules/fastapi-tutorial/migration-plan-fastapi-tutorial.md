---
source-path: cookbooks/fastapi-tutorial
---

# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook deploys the `fastapi-tutorial` FastAPI application from `https://github.com/dibanez/fastapi_tutorial.git` at revision `main`. It installs Python, Git, PostgreSQL, and development dependencies; creates `/opt/fastapi-tutorial` and its virtual environment; installs Python requirements; initializes the PostgreSQL `fastapi` user and `fastapi_db` database; writes an application environment file; creates a systemd unit; and enables and starts the application on port `8000`. The PostgreSQL password is hardcoded in the Chef recipe and must be replaced with an Ansible Vault variable or AAP-managed secret.

## Service Type and Instances

**Service Type**: Application Server with PostgreSQL database dependency

**Configured Instances**:

- **fastapi-tutorial**: FastAPI application served by Uvicorn
  - Repository: `https://github.com/dibanez/fastapi_tutorial.git`
  - Revision: `main`
  - Location/Path: `/opt/fastapi-tutorial`
  - Virtual environment: `/opt/fastapi-tutorial/venv`
  - Working directory: `/opt/fastapi-tutorial`
  - Startup command: `/opt/fastapi-tutorial/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000`
  - Listen address: `0.0.0.0`
  - Port: `8000`
  - Service user: `root`
  - Restart policy: `always`
  - Environment file: `/opt/fastapi-tutorial/.env`
  - systemd unit: `/etc/systemd/system/fastapi-tutorial.service`

- **postgresql**: PostgreSQL database service used by the FastAPI application
  - Service name: `postgresql`
  - Host: `localhost`
  - Port: Distribution default, normally `5432`
  - Database: `fastapi_db`
  - Database user: `fastapi`
  - Database owner: `fastapi`
  - Data directory: Distribution-managed; not configured by the cookbook
  - Cluster/version: Not explicitly configured

## File Structure

**MANDATORY: Preserve this section from the original plan.**

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

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/fastapi-tutorial/recipes/default.rb`):
   - Installs the system packages `python3`, `python3-pip`, `python3-venv`, `git`, `postgresql`, `postgresql-contrib`, and `libpq-dev`.
   - The package versions are not pinned.
   - The cookbook metadata supports Ubuntu `>= 18.04` and CentOS `>= 7.0`; package names must be mapped appropriately for the target distribution, particularly `python3-venv`, `libpq-dev`, and PostgreSQL development packages.
   - Creates `/opt/fastapi-tutorial` recursively with owner `root`, group `root`, and mode `0755`.
   - Synchronizes `https://github.com/dibanez/fastapi_tutorial.git` into `/opt/fastapi-tutorial` at revision `main`.
   - Creates the Python virtual environment with:
     ```bash
     python3 -m venv /opt/fastapi-tutorial/venv
     ```
     using `/opt/fastapi-tutorial/venv` as the `creates` guard.
   - Installs Python dependencies from `/opt/fastapi-tutorial/requirements.txt` using:
     ```bash
     /opt/fastapi-tutorial/venv/bin/pip install -r /opt/fastapi-tutorial/requirements.txt
     ```
     with `/opt/fastapi-tutorial` as the working directory. This command runs on every Chef convergence because it has no guard.
   - Enables and starts the `postgresql` service.
   - Attempts to create the PostgreSQL user and database:
     - User: `fastapi`
     - Password: `fastapi_password`
     - Database: `fastapi_db`
     - Owner: `fastapi`
     - Privileges: `ALL`
   - The database initialization commands use `|| true`, which suppresses both already-exists conditions and genuine errors. The Ansible migration should use `community.postgresql.postgresql_user`, `community.postgresql.postgresql_db`, and `community.postgresql.postgresql_privs` for idempotence.
   - Creates the inline Chef `file` resource `/opt/fastapi-tutorial/.env` with owner `root`, group `root`, and mode `0644`. Its content is:
     ```dotenv
     PROJECT_NAME="FastAPI Tutorial"
     API_VERSION=1.0.0
     DATABASE_URL=postgresql://fastapi:fastapi_password@localhost/fastapi_db
     ```
     The Ansible implementation should preferably use mode `0600` to protect the database password.
   - Creates the inline Chef `file` resource `/etc/systemd/system/fastapi-tutorial.service` with owner `root`, group `root`, and mode `0644`. Its unit content is:
     ```ini
     [Unit]
     Description=FastAPI Tutorial Service
     After=network.target postgresql.service

     [Service]
     Type=simple
     User=root
     WorkingDirectory=/opt/fastapi-tutorial
     Environment="PATH=/opt/fastapi-tutorial/venv/bin"
     ExecStart=/opt/fastapi-tutorial/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000
     Restart=always

     [Install]
     WantedBy=multi-user.target
     ```
   - Notifies `execute[systemd_reload]` immediately when the systemd unit file changes.
   - Runs `systemctl daemon-reload` through `execute[systemd_reload]`; the resource has action `nothing` and runs only after notification.
   - Enables and starts the `fastapi-tutorial` systemd service.

**Iterations**: None. The cookbook contains no `.each` loops or collection-based instance iterations.

## Dependencies

**External cookbook dependencies**: None declared in `metadata.rb`.

**System package dependencies**:
- `python3`
- `python3-pip`
- `python3-venv`
- `git`
- `postgresql`
- `postgresql-contrib`
- `libpq-dev`

**Python dependency source**:
- `/opt/fastapi-tutorial/requirements.txt`

**Source repository**: `https://github.com/dibanez/fastapi_tutorial.git`

**Repository revision**: `main`

**Service dependencies**:
- `network.target`
- `postgresql`

**Managed services**:
- `postgresql`
- `fastapi-tutorial`

## Credentials

**Detection Summary**: 1 database credential detected in 1 recipe file.

**Source**:
- **Provider**: Hardcoded in the Chef recipe
- **URL**: None
- **Path**: None

### PostgreSQL password for the `fastapi` user

- **Variable(s)**:
  - No Chef attribute variable is used.
  - Literal password: `fastapi_password`
  - Database URL reference: `DATABASE_URL=postgresql://fastapi:fastapi_password@localhost/fastapi_db`
- **Source file(s)**:
  - `cookbooks/fastapi-tutorial/recipes/default.rb`
- **Current storage**: Hardcoded in the PostgreSQL creation command and inline `.env` file content
- **Usage context**:
  - Creates the PostgreSQL user `fastapi`
  - Authenticates the FastAPI application to PostgreSQL through `DATABASE_URL`
- **Migration requirement**:
  - Store the password in Ansible Vault or an AAP credential/secret.
  - Use the secret for both the PostgreSQL user task and generated `.env` file.
  - Do not expose the secret through command-line arguments, task output, or logs.
  - Use `.env` mode `0600` unless exact compatibility with the Chef mode `0644` is required.

No Chef data bags, encrypted data bags, Chef Vault calls, CyberArk integration, Conjur variables, Vault attributes, environment-variable lookups, TLS certificate references, API keys, or tokens were detected.

## Checks for the Migration

**Files to verify**:
- `/opt/fastapi-tutorial`
- `/opt/fastapi-tutorial/requirements.txt`
- `/opt/fastapi-tutorial/venv`
- `/opt/fastapi-tutorial/.env`
- `/etc/systemd/system/fastapi-tutorial.service`

PostgreSQL data and log locations are distribution-managed and are not explicitly configured by this cookbook. Verify active locations through the PostgreSQL service and operating-system configuration.

**Service endpoints to check**:
- FastAPI instance `fastapi-tutorial`: `http://127.0.0.1:8000`
- FastAPI listen address: `0.0.0.0`
- FastAPI port: `8000`
- PostgreSQL instance `postgresql`: `localhost:5432`, normally
- PostgreSQL database: `fastapi_db`
- PostgreSQL user: `fastapi`

**Templates rendered**:
- None. No Chef templates or Ansible templates are referenced.
- `/opt/fastapi-tutorial/.env`: inline Chef `file` resource, not a template
- `/etc/systemd/system/fastapi-tutorial.service`: inline Chef `file` resource, not a template

## Pre-flight checks

```bash
# Check the fastapi-tutorial instance
systemctl status fastapi-tutorial
systemctl is-enabled fastapi-tutorial
systemctl is-active fastapi-tutorial
systemctl show fastapi-tutorial --property=LoadState,ActiveState,SubState,UnitFileState

# Check the postgresql instance
systemctl status postgresql
systemctl is-enabled postgresql
systemctl is-active postgresql

# Check the application directory and repository
ls -ld /opt/fastapi-tutorial
git -C /opt/fastapi-tutorial remote -v
git -C /opt/fastapi-tutorial branch --show-current
git -C /opt/fastapi-tutorial rev-parse HEAD

# Check the Python virtual environment
test -x /opt/fastapi-tutorial/venv/bin/python
test -x /opt/fastapi-tutorial/venv/bin/pip
/opt/fastapi-tutorial/venv/bin/python --version
/opt/fastapi-tutorial/venv/bin/pip --version

# Check Python requirements
test -f /opt/fastapi-tutorial/requirements.txt
/opt/fastapi-tutorial/venv/bin/pip check

# Check the application environment file without printing the secret
stat /opt/fastapi-tutorial/.env
grep -E '^(PROJECT_NAME|API_VERSION|DATABASE_URL)=' /opt/fastapi-tutorial/.env

# Check the fastapi-tutorial systemd unit
systemctl cat fastapi-tutorial
grep -E '^(Description|After|Type|User|WorkingDirectory|Environment|ExecStart|Restart|WantedBy)=' \
  /etc/systemd/system/fastapi-tutorial.service

# Reload and validate systemd unit definitions
systemctl daemon-reload
systemctl show fastapi-tutorial --property=LoadState,ActiveState,SubState,UnitFileState

# Check the postgresql instance readiness
pg_isready -h localhost -p 5432

# Check the PostgreSQL user
sudo -u postgres psql -d postgres -c \
  "SELECT usename FROM pg_catalog.pg_user WHERE usename = 'fastapi';"

# Check the PostgreSQL database and owner
sudo -u postgres psql -d postgres -c \
  "SELECT datname, pg_catalog.pg_get_userbyid(datdba) AS owner
   FROM pg_catalog.pg_database WHERE datname = 'fastapi_db';"

# Verify application database connectivity.
# Supply FASTAPI_DB_PASSWORD securely without logging or exposing its value.
read -r -s -p "Database password: " FASTAPI_DB_PASSWORD
echo
PGPASSWORD="$FASTAPI_DB_PASSWORD" psql \
  -h localhost \
  -U fastapi \
  -d fastapi_db \
  -c "SELECT current_database(), current_user;"
unset FASTAPI_DB_PASSWORD

# Check the fastapi-tutorial listening port
ss -tlnp | grep ':8000'
netstat -tulpn 2>/dev/null | grep ':8000'

# Check the postgresql listening port
ss -tlnp | grep ':5432'
netstat -tulpn 2>/dev/null | grep ':5432'

# Check HTTP access to the fastapi-tutorial instance
curl -i http://127.0.0.1:8000/

# Check the FastAPI process
ps aux | grep '[u]vicorn'
pgrep -af '/opt/fastapi-tutorial/venv/bin/uvicorn'

# Inspect fastapi-tutorial logs
journalctl -u fastapi-tutorial --no-pager -n 100

# Inspect postgresql logs
journalctl -u postgresql --no-pager -n 100
```

The HTTP check should confirm that Uvicorn is listening on port `8000` and returning an HTTP response. The exact response code for `/` depends on the checked-out FastAPI application and is not defined by this cookbook.