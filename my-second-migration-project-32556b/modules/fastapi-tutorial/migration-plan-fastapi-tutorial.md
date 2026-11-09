---
source-path: cookbooks/fastapi-tutorial
---

# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook deploys the `fastapi-tutorial` FastAPI application to `/opt/fastapi-tutorial`, installs Python, PostgreSQL, Git, and PostgreSQL development dependencies, creates a Python virtual environment, installs application requirements, provisions the PostgreSQL `fastapi` role and `fastapi_db` database, writes an environment file and systemd unit, and enables and starts PostgreSQL and the FastAPI service. The application port, Git repository details, environment-file contents, and systemd unit contents are not exposed in the supplied execution tree and must be confirmed from the source recipe or deployment variables.

## Service Type and Instances

**Service Type**: Application Server with PostgreSQL database dependency

**Configured Instances**:

- **fastapi-tutorial**: FastAPI application service
  - Location/Path: `/opt/fastapi-tutorial`
  - Virtual environment: `/opt/fastapi-tutorial/venv`
  - Requirements file: `/opt/fastapi-tutorial/requirements.txt`
  - Environment file: `/opt/fastapi-tutorial/.env`
  - Systemd unit: `/etc/systemd/system/fastapi-tutorial.service`
  - Service name: `fastapi-tutorial`
  - Database user: `fastapi`
  - Database name: `fastapi_db`
  - Database owner: `fastapi`
  - Database password: `fastapi_password`, currently hardcoded by Chef and requiring migration to Ansible Vault or an approved secret provider
  - Port/Socket: Application port is not specified in the supplied execution tree. Obtain it from the deployed systemd unit or application configuration before creating the Ansible health check.

- **postgresql**: PostgreSQL database service for `fastapi-tutorial`
  - Service name: `postgresql`
  - Database role: `fastapi`
  - Database: `fastapi_db`
  - Port/Socket: Not specified in the supplied execution tree. Verify the active PostgreSQL listener and Unix socket on the target host.

**Iterations**: None. The execution tree contains no collection iteration or `.each` loop.

## File Structure

**MANDATORY: Preserve this section from the original plan.**

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

The `.env` file and systemd unit are generated inline by Chef `file` resources. They are not cookbook static files or templates in the supplied execution tree.

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/fastapi-tutorial/recipes/default.rb`):
   - Installs the required operating-system packages with a `package` resource:
     - `python3`
     - `python3-pip`
     - `python3-venv`
     - `git`
     - `postgresql`
     - `postgresql-contrib`
     - `libpq-dev`
   - Creates `/opt/fastapi-tutorial` with mode `0755`.
   - Synchronizes the application source into `/opt/fastapi-tutorial` using a `git` resource with action `sync`.
   - The repository URL, revision, branch, and checkout user are not exposed and must be obtained from the actual Chef recipe or migration variables.
   - Creates the virtual environment with:
     ```bash
     python3 -m venv /opt/fastapi-tutorial/venv
     ```
   - Installs Python dependencies from `/opt/fastapi-tutorial/requirements.txt` using:
     ```bash
     /opt/fastapi-tutorial/venv/bin/pip install -r /opt/fastapi-tutorial/requirements.txt
     ```
   - Enables and starts the `postgresql` service.
   - Executes three PostgreSQL SQL statements as the `postgres` administrative user:
     1. `CREATE USER fastapi WITH PASSWORD 'fastapi_password';`
     2. `CREATE DATABASE fastapi_db OWNER fastapi;`
     3. `GRANT ALL PRIVILEGES ON DATABASE fastapi_db TO fastapi;`
   - The original Chef commands append `|| true`, suppressing errors including already-existing objects. The Ansible migration should instead use idempotent PostgreSQL modules:
     - `community.postgresql.postgresql_user` for role `fastapi`
     - `community.postgresql.postgresql_db` for database `fastapi_db` owned by `fastapi`
     - `community.postgresql.postgresql_privs` for database privileges
   - Creates `/opt/fastapi-tutorial/.env` with mode `0644` using inline Chef file content. The content is not exposed and must be obtained from the actual recipe.
   - Creates `/etc/systemd/system/fastapi-tutorial.service` with mode `0644` using inline Chef file content. The unit contents are not exposed and must be obtained from the actual recipe.
   - The systemd unit content must be inspected to determine the startup command, runtime user, working directory, environment-file usage, listening address and port, restart behavior, and service dependencies.
   - Declares `execute[systemd_reload]` with command `systemctl daemon-reload` and action `nothing`. No notification relationship is shown in the supplied execution tree, so Chef does not explicitly execute this command during the effective flow.
   - Ansible should normally reload systemd after writing or changing the unit and before starting `fastapi-tutorial`.
   - Enables and starts the `fastapi-tutorial` service.
   - Iterations: None.

**Resource summary**:

- `package`: 1 resource installing 7 packages
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

No standalone group creation, custom resources, providers, attribute files, templates, or static cookbook files are present in the supplied execution tree. Ownership and group settings are not provided by the structured execution tree and must be confirmed from the actual Chef recipe before migration.

## Dependencies

**External cookbook dependencies**: None identified in the supplied metadata or execution tree.

**System package dependencies**:

- `python3`
- `python3-pip`
- `python3-venv`
- `git`
- `postgresql`
- `postgresql-contrib`
- `libpq-dev`

**Service dependencies**:

- `postgresql`: Must be enabled and started before database initialization.
- `fastapi-tutorial`: Must be started after the application checkout, virtual environment, dependencies, database objects, environment file, and systemd unit are available.

**Source-code dependencies**:

- A Git repository is synchronized into `/opt/fastapi-tutorial`.
- The repository URL and revision are not present in the supplied execution tree and must be obtained from the Chef recipe or supplied as migration variables.

## Credentials

**Detection Summary**: 1 hard-coded database credential detected in the supplied execution tree. The inline `.env` content is not exposed and may contain additional sensitive values that cannot be enumerated from the supplied analysis.

**Source**:

- **Provider**: Hardcoded in a Chef `execute` command
- **URL**: None detected
- **Path**: PostgreSQL database user creation command
- **Secret**: `fastapi_password`

### PostgreSQL database password

- **Variable(s)**: No named Chef variable is shown; the literal password is `fastapi_password`.
- **Source file(s)**: `cookbooks/fastapi-tutorial/recipes/default.rb`
- **Current storage**: Hardcoded plaintext in the `execute[create_db_user]` command.
- **Usage context**: Password for PostgreSQL role `fastapi`, which owns and accesses database `fastapi_db`.
- **Ansible handling**: Store the password in Ansible Vault or an approved external secret provider. Use the vaulted value with `community.postgresql.postgresql_user` and in the generated `.env` file if required by the application.

### `.env` file contents

- **Variable(s)**: Not available in the supplied execution tree.
- **Source file(s)**: `cookbooks/fastapi-tutorial/recipes/default.rb`
- **Current storage**: Inline Chef file content.
- **Usage context**: Application environment configuration.
- **Credential status**: The supplied tree does not expose whether the file contains passwords, tokens, database URLs, or other secrets.
- **Ansible handling**: Inspect the actual Chef heredoc before migration. Represent all sensitive values with Ansible Vault or an approved secret provider and do not commit them as plaintext.

No data bags, encrypted data bags, Chef Vault calls, CyberArk or Conjur integrations, ERB template environment-variable references, TLS certificate references, API keys, or visible embedded database connection credentials were identified in the supplied analysis.

## Checks for the Migration

**Files to verify**:

- `cookbooks/fastapi-tutorial/recipes/default.rb`
- `/opt/fastapi-tutorial`
- `/opt/fastapi-tutorial/requirements.txt`
- `/opt/fastapi-tutorial/venv`
- `/opt/fastapi-tutorial/.env`
- `/etc/systemd/system/fastapi-tutorial.service`

**Database objects to verify**:

- PostgreSQL role: `fastapi`
- PostgreSQL database: `fastapi_db`
- Database owner: `fastapi`
- Database privileges: `fastapi` has all privileges on `fastapi_db`

**Service endpoints to check**:

- PostgreSQL network port: Not specified; verify the active PostgreSQL listener.
- PostgreSQL Unix socket: Not specified; inspect the active PostgreSQL configuration if required.
- `fastapi-tutorial` application port: Not specified; obtain it from `/etc/systemd/system/fastapi-tutorial.service` or application configuration before testing.

**Templates rendered**:

- None.

**Inline files rendered**:

- `/opt/fastapi-tutorial/.env`: rendered once by the Chef `file` resource.
- `/etc/systemd/system/fastapi-tutorial.service`: rendered once by the Chef `file` resource.

## Pre-flight checks

```bash
# Confirm required packages are installed
dpkg -l python3 python3-pip python3-venv git postgresql postgresql-contrib libpq-dev

# Confirm the fastapi-tutorial application checkout exists
ls -ld /opt/fastapi-tutorial
test -f /opt/fastapi-tutorial/requirements.txt
test -x /opt/fastapi-tutorial/venv/bin/python

# Confirm the fastapi-tutorial virtual environment can execute Python
/opt/fastapi-tutorial/venv/bin/python --version

# Confirm the fastapi-tutorial environment file exists
ls -l /opt/fastapi-tutorial/.env

# Check the postgresql service individually
systemctl status postgresql --no-pager
systemctl is-enabled postgresql
systemctl is-active postgresql

# Verify the PostgreSQL fastapi role
sudo -u postgres psql -tAc \
  "SELECT rolname FROM pg_roles WHERE rolname = 'fastapi';"

# Verify the fastapi_db database and owner
sudo -u postgres psql -tAc \
  "SELECT datname, pg_catalog.pg_get_userbyid(datdba) FROM pg_database WHERE datname = 'fastapi_db';"

# Verify that the fastapi role can connect to fastapi_db
sudo -u postgres psql -d fastapi_db \
  -c "SELECT current_database(), current_user;"

# Verify fastapi_db privileges
sudo -u postgres psql -c "\l+ fastapi_db"

# Verify the PostgreSQL listener and Unix socket
ss -ltnp | grep postgres || true
ss -lx | grep postgres || true

# Check the fastapi-tutorial unit file
test -f /etc/systemd/system/fastapi-tutorial.service

# Reload systemd metadata after deploying the unit
systemctl daemon-reload

# Check the fastapi-tutorial service individually
systemctl status fastapi-tutorial --no-pager
systemctl is-enabled fastapi-tutorial
systemctl is-active fastapi-tutorial

# Inspect the effective fastapi-tutorial unit configuration
systemctl cat fastapi-tutorial

# Inspect the fastapi-tutorial startup command and runtime settings
systemctl show fastapi-tutorial \
  -p ExecStart \
  -p WorkingDirectory \
  -p Environment \
  -p EnvironmentFiles

# Review fastapi-tutorial logs
journalctl -u fastapi-tutorial --no-pager -n 100

# Verify application paths and requested modes
ls -ld /opt/fastapi-tutorial
ls -ld /opt/fastapi-tutorial/venv
stat -c '%a %n' /opt/fastapi-tutorial
stat -c '%a %n' /opt/fastapi-tutorial/.env
stat -c '%a %n' /etc/systemd/system/fastapi-tutorial.service

# Validate the fastapi-tutorial systemd unit syntax
systemd-analyze verify /etc/systemd/system/fastapi-tutorial.service

# Review recent fastapi-tutorial logs
journalctl -u fastapi-tutorial --since "-15 minutes" --no-pager
```

The FastAPI application port is not exposed by the execution tree. After inspecting the deployed unit and identifying the configured port, run:

```bash
# Replace <FASTAPI_PORT> with the port from the deployed fastapi-tutorial unit
ss -ltnp | grep '<FASTAPI_PORT>'

# Replace /health with the actual endpoint if defined by the application
curl -i "http://127.0.0.1:<FASTAPI_PORT>/health"
```

Expected results:

- `postgresql` is enabled and active.
- PostgreSQL role `fastapi` exists.
- Database `fastapi_db` exists and is owned by `fastapi`.
- The `fastapi` role can connect to `fastapi_db`.
- The `fastapi-tutorial` unit exists, validates successfully, is enabled, and is active.
- The FastAPI process runs from `/opt/fastapi-tutorial/venv`.
- The configured application listener is bound to the expected address and port.
- The application health endpoint returns a successful response if one is defined.
