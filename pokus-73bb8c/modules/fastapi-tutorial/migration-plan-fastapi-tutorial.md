---
source-path: cookbooks/fastapi-tutorial
---

# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook deploys the FastAPI application from the `main` branch of `https://github.com/dibanez/fastapi_tutorial.git` into `/opt/fastapi-tutorial`, installs Python, Git, PostgreSQL, and PostgreSQL client dependencies, creates a Python virtual environment, installs application requirements, provisions the `fastapi` PostgreSQL user and `fastapi_db` database, writes the application environment configuration, and manages the application with a systemd service listening on `0.0.0.0:8000`. The cookbook configures one FastAPI application instance and one PostgreSQL database instance.

## Service Type and Instances

**Service Type**: Application Server with PostgreSQL database dependency

**Configured Instances**:

- **fastapi-tutorial**: FastAPI application served by Uvicorn
  - Location/Path: `/opt/fastapi-tutorial`
  - Source repository: `https://github.com/dibanez/fastapi_tutorial.git`
  - Git revision: `main`
  - Python virtual environment: `/opt/fastapi-tutorial/venv`
  - Port/Socket: TCP `0.0.0.0:8000`
  - Application command: `/opt/fastapi-tutorial/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000`
  - Working directory: `/opt/fastapi-tutorial`
  - Service user: `root`
  - Systemd unit: `/etc/systemd/system/fastapi-tutorial.service`
  - Restart policy: `Restart=always`
  - Dependencies: `network.target` and `postgresql.service`

- **PostgreSQL instance**: Application database service
  - Service: `postgresql`
  - Connection host: `localhost`
  - Port/Socket: Platform-default PostgreSQL listener; no custom port or Unix socket is configured
  - Database user: `fastapi`
  - Database password: `fastapi_password`
  - Database: `fastapi_db`
  - Database owner: `fastapi`
  - Privileges: `fastapi` receives all privileges on `fastapi_db`

**Iterations**: None. The recipe contains no collection iteration or `.each` loop.
Something new


## File Structure

**MANDATORY: Preserve this section from the original plan.**

**Recipes:**
```text
recipes/default.rb
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

The cookbook writes `.env` and the systemd unit directly using Chef `file` resources. It does not render ERB templates or deploy static files through `cookbook_file` or `remote_file`.

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/fastapi-tutorial/recipes/default.rb`):
   - Installs the seven operating-system packages `python3`, `python3-pip`, `python3-venv`, `git`, `postgresql`, `postgresql-contrib`, and `libpq-dev` using one `package` resource.
   - Creates `/opt/fastapi-tutorial` with owner `root`, group `root`, mode `0755`, and recursive creation enabled using `directory`.
   - Synchronizes `https://github.com/dibanez/fastapi_tutorial.git` at revision `main` into `/opt/fastapi-tutorial` using `git`.
   - Creates the virtual environment with `python3 -m venv /opt/fastapi-tutorial/venv` using `execute[create_venv]`.
   - Runs the virtual-environment creation only when `/opt/fastapi-tutorial/venv` does not exist through a `creates` guard.
   - Installs requirements with `/opt/fastapi-tutorial/venv/bin/pip install -r /opt/fastapi-tutorial/requirements.txt`, using `/opt/fastapi-tutorial` as the working directory, through `execute[install_dependencies]`.
   - Enables and starts the `postgresql` service using `service[postgresql]`.
   - Runs the following PostgreSQL initialization commands as the PostgreSQL operating-system user through `execute[create_db_user]`:
     - Creates user `fastapi` with password `fastapi_password`.
     - Creates database `fastapi_db` owned by `fastapi`.
     - Grants all privileges on `fastapi_db` to `fastapi`.
   - The original commands use `|| true` to tolerate existing users and databases. The migration should use idempotent `community.postgresql.postgresql_user`, `community.postgresql.postgresql_db`, and, if needed, `community.postgresql.postgresql_privs` tasks executed as the PostgreSQL administrative account.
   - Creates `/opt/fastapi-tutorial/.env` using `file[/opt/fastapi-tutorial/.env]` with owner `root`, group `root`, and mode `0644`.
   - Writes the following environment values:
     ```dotenv
     PROJECT_NAME="FastAPI Tutorial"
     API_VERSION=1.0.0
     DATABASE_URL=postgresql://fastapi:fastapi_password@localhost/fastapi_db
     ```
   - Creates `/etc/systemd/system/fastapi-tutorial.service` using `file[/etc/systemd/system/fastapi-tutorial.service]` with owner `root`, group `root`, and mode `0644`.
   - Writes a systemd unit with `User=root`, `WorkingDirectory=/opt/fastapi-tutorial`, `Environment="PATH=/opt/fastapi-tutorial/venv/bin"`, Uvicorn entry point `app.main:app`, bind address `0.0.0.0`, port `8000`, `Restart=always`, and ordering after `network.target` and `postgresql.service`.
   - Immediately notifies `execute[systemd_reload]` when `/etc/systemd/system/fastapi-tutorial.service` changes.
   - Runs `systemctl daemon-reload` through `execute[systemd_reload]`, which is declared with `action :nothing` and executes only when notified.
   - Enables and starts the `fastapi-tutorial` service using `service[fastapi-tutorial]` after the unit file has been written and systemd has been reloaded.

**Resource summary** for `cookbooks/fastapi-tutorial/recipes/default.rb`:

- `package`: 1 resource installing 7 packages
- `directory`: 1 resource
- `git`: 1 resource
- `execute`: 4 resources:
  - `create_venv`
  - `install_dependencies`
  - `create_db_user`
  - `systemd_reload`
- `file`: 2 resources:
  - `/opt/fastapi-tutorial/.env`
  - `/etc/systemd/system/fastapi-tutorial.service`
- `service`: 2 resources:
  - `postgresql`
  - `fastapi-tutorial`

## Dependencies

**External cookbook dependencies**: None declared in the provided metadata.

**Supported platforms**:

- Ubuntu `>= 18.04`
- CentOS `>= 7.0`

**System package dependencies**:

- `python3`
- `python3-pip`
- `python3-venv`
- `git`
- `postgresql`
- `postgresql-contrib`
- `libpq-dev`

**Service dependencies**:

- `postgresql`
- `fastapi-tutorial`

**Application dependencies**:

- Python requirements from `/opt/fastapi-tutorial/requirements.txt`
- Requirements installed into `/opt/fastapi-tutorial/venv`

## Credentials

**Detection Summary**: 1 database credential detected in 1 recipe file.

**Source**:

- **Provider**: Hardcoded in Chef recipe
- **URL**: None
- **Path**: None

### PostgreSQL `fastapi` Database Password

- **Variable(s)**:
  - PostgreSQL password: `fastapi_password`
  - Embedded connection string: `postgresql://fastapi:fastapi_password@localhost/fastapi_db`
- **Source file(s)**:
  - `cookbooks/fastapi-tutorial/recipes/default.rb`
- **Current storage**: Hardcoded in the recipe and duplicated in `/opt/fastapi-tutorial/.env`
- **Usage context**:
  - Sets the password for PostgreSQL user `fastapi`
  - Authenticates the FastAPI application to database `fastapi_db`
- **Migration handling**:
  - Store the password in Ansible Vault or an approved external secret-management system.
  - Use the protected value when creating the PostgreSQL user.
  - Construct `DATABASE_URL` from the protected value rather than committing `fastapi_password` in plaintext.

No data bag, Chef Vault, CyberArk, Conjur, environment-variable secret lookup, TLS certificate, private key, API token, or external vault endpoint was detected.

## Checks for the Migration

**Files to verify**:

- `/opt/fastapi-tutorial`
- `/opt/fastapi-tutorial/venv`
- `/opt/fastapi-tutorial/requirements.txt`
- `/opt/fastapi-tutorial/.env`
- `/etc/systemd/system/fastapi-tutorial.service`

**Database objects to verify**:

- PostgreSQL user `fastapi`
- PostgreSQL database `fastapi_db`
- Database owner `fastapi`
- Database host `localhost`

**Service endpoints to check**:

- FastAPI: `0.0.0.0:8000`
- PostgreSQL: platform-default local PostgreSQL listener
- Unix sockets: no custom Unix socket configured

**Templates rendered**:

- None; no templates are used or rendered.

## Pre-flight checks:

```bash
# Verify required packages
dpkg -l python3 python3-pip python3-venv git postgresql postgresql-contrib libpq-dev

# Verify the fastapi-tutorial application directory
stat -c '%A %U:%G %n' /opt/fastapi-tutorial

# Verify the fastapi-tutorial repository checkout
git -C /opt/fastapi-tutorial status
git -C /opt/fastapi-tutorial branch --show-current

# Verify the fastapi-tutorial virtual environment
test -x /opt/fastapi-tutorial/venv/bin/python
/opt/fastapi-tutorial/venv/bin/python --version

# Verify fastapi-tutorial Python dependencies
/opt/fastapi-tutorial/venv/bin/pip check

# Verify the PostgreSQL service instance
systemctl status postgresql
systemctl is-enabled postgresql

# Verify PostgreSQL user fastapi
sudo -u postgres psql -tAc \
  "SELECT usename FROM pg_user WHERE usename = 'fastapi';"

# Verify PostgreSQL database fastapi_db and owner fastapi
sudo -u postgres psql -tAc \
  "SELECT datname, pg_catalog.pg_get_userbyid(datdba) FROM pg_database WHERE datname = 'fastapi_db';"

# Verify the fastapi application database connection
PGPASSWORD='fastapi_password' psql \
  -h localhost \
  -U fastapi \
  -d fastapi_db \
  -c "SELECT current_user, current_database();"

# Verify the fastapi application database can execute queries
PGPASSWORD='fastapi_password' psql \
  -h localhost \
  -U fastapi \
  -d fastapi_db \
  -c "SELECT 1;"

# Verify environment file ownership and mode
stat -c '%A %U:%G %n' /opt/fastapi-tutorial/.env

# Verify non-secret environment values
grep -E '^(PROJECT_NAME|API_VERSION)=' /opt/fastapi-tutorial/.env

# Verify DATABASE_URL exists without exposing its value
grep -q '^DATABASE_URL=' /opt/fastapi-tutorial/.env

# Verify the fastapi-tutorial systemd unit ownership and mode
stat -c '%A %U:%G %n' /etc/systemd/system/fastapi-tutorial.service

# Verify required fastapi-tutorial unit settings
grep -E '^(Description|After|Type|User|WorkingDirectory|Environment|ExecStart|Restart|WantedBy)=' \
  /etc/systemd/system/fastapi-tutorial.service

# Reload systemd and verify the fastapi-tutorial service instance
systemctl daemon-reload
systemctl is-enabled fastapi-tutorial
systemctl status fastapi-tutorial

# Verify the fastapi-tutorial service process
ps aux | grep '[u]vicorn app.main:app'

# Verify the fastapi-tutorial TCP listener
ss -tlnp | grep ':8000'
lsof -iTCP:8000 -sTCP:LISTEN

# Test the fastapi-tutorial HTTP endpoint
curl -i http://127.0.0.1:8000/

# Review fastapi-tutorial service logs
journalctl -u fastapi-tutorial --no-pager -n 100

# Confirm fastapi-tutorial ordering after PostgreSQL
systemctl cat fastapi-tutorial | grep 'After=.*postgresql.service'

# Verify the FastAPI executable
test -x /opt/fastapi-tutorial/venv/bin/uvicorn
/opt/fastapi-tutorial/venv/bin/uvicorn --version

# Validate the fastapi-tutorial service restart
systemctl restart fastapi-tutorial
systemctl is-active fastapi-tutorial
curl -i http://127.0.0.1:8000/
```

Do not print the complete `/opt/fastapi-tutorial/.env` file in shared logs because it contains the PostgreSQL password. The FastAPI service must listen on TCP port `8000`; the root HTTP request should return an application response rather than a connection-refused error. The cookbook does not define a health-check route, so no `/health` or `/status` endpoint should be assumed.