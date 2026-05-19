---
source-path: cookbooks/fastapi-tutorial
---

# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook deploys a FastAPI Python web application with a PostgreSQL database backend. It sets up a single instance of the application that runs on port 8000, with a dedicated database and systemd service.

## Service Type and Instances

**Service Type**: Web Server (FastAPI Python application)

**Configured Instances**:

- **fastapi-tutorial**: Python FastAPI web application
  - Location/Path: /opt/fastapi-tutorial
  - Port/Socket: 8000
  - Key Config: 
    - Database URL: postgresql://fastapi:fastapi_password@localhost/fastapi_db
    - Project Name: "FastAPI Tutorial"
    - API Version: 1.0.0

## File Structure

```
cookbooks/fastapi-tutorial/recipes/default.rb
cookbooks/fastapi-tutorial/metadata.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/fastapi-tutorial/recipes/default.rb`):
   - Installs Python 3 and required system packages: python3, python3-pip, python3-venv, git, postgresql, postgresql-contrib, libpq-dev
   - Creates application directory: /opt/fastapi-tutorial
   - Clones FastAPI tutorial repository from https://github.com/dibanez/fastapi_tutorial.git (branch: main)
   - Creates Python virtual environment at /opt/fastapi-tutorial/venv
   - Installs Python dependencies from requirements.txt
   - Configures PostgreSQL service (enable and start)
   - Creates database user 'fastapi' with password 'fastapi_password'
   - Creates database 'fastapi_db' owned by 'fastapi' user
   - Grants all privileges on database to 'fastapi' user
   - Creates environment configuration file at /opt/fastapi-tutorial/.env with database connection details
   - Creates systemd service file at /etc/systemd/system/fastapi-tutorial.service
   - Reloads systemd daemon when service file changes
   - Enables and starts the fastapi-tutorial service
   - Resources: package (1), directory (1), git (1), execute (3), service (2), file (2)

## Dependencies

**External cookbook dependencies**: None specified in metadata.rb
**System package dependencies**: python3, python3-pip, python3-venv, git, postgresql, postgresql-contrib, libpq-dev
**Service dependencies**: postgresql.service (required by fastapi-tutorial.service)

## Credentials

**Detection Summary**: 2 credentials detected across 2 files

**Source**:
  - **Provider**: Hardcoded
  - **URL**: N/A
  - **Path**: N/A

### Database Password

- **Variable(s)**: 'fastapi_password'
- **Source file(s)**: cookbooks/fastapi-tutorial/recipes/default.rb
- **Current storage**: hardcoded
- **Usage context**: PostgreSQL database user password, used for database connection

### Database Connection String

- **Variable(s)**: 'DATABASE_URL'
- **Source file(s)**: cookbooks/fastapi-tutorial/recipes/default.rb
- **Current storage**: hardcoded in .env file
- **Usage context**: FastAPI application database connection string

## Checks for the Migration

**Files to verify**:
- /opt/fastapi-tutorial (application directory)
- /opt/fastapi-tutorial/venv (Python virtual environment)
- /opt/fastapi-tutorial/.env (environment configuration)
- /etc/systemd/system/fastapi-tutorial.service (systemd service file)

**Service endpoints to check**:
- Ports listening: 8000
- Network interfaces: 0.0.0.0 (all interfaces)

**Templates rendered**:
- No templates used, direct file resources with inline content

## Pre-flight checks:
```bash
# Service status
systemctl status fastapi-tutorial
systemctl status postgresql
ps aux | grep uvicorn

# Application health
curl -I http://localhost:8000/
curl -I http://localhost:8000/docs  # FastAPI Swagger documentation

# Database connectivity
sudo -u postgres psql -c "\l" | grep fastapi_db
sudo -u postgres psql -c "\du" | grep fastapi
PGPASSWORD=fastapi_password psql -h localhost -U fastapi -d fastapi_db -c "SELECT 1;"

# Configuration validation
cat /opt/fastapi-tutorial/.env
cat /etc/systemd/system/fastapi-tutorial.service
ls -la /opt/fastapi-tutorial/venv/bin/  # Check if virtual environment is properly set up

# Python dependencies
/opt/fastapi-tutorial/venv/bin/pip list | grep -E 'fastapi|uvicorn|sqlalchemy'

# Logs
journalctl -u fastapi-tutorial -f
journalctl -u postgresql -f

# Network listening
netstat -tulpn | grep 8000
ss -tlnp | grep 8000
lsof -i :8000

# Git repository
cd /opt/fastapi-tutorial && git remote -v
cd /opt/fastapi-tutorial && git branch

# Process resources
ps aux | grep uvicorn
systemctl show fastapi-tutorial | grep -E 'ExecStart|WorkingDirectory|Environment'
```