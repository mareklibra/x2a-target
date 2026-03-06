# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook deploys a FastAPI Python web application with a PostgreSQL database backend. It sets up a single instance of the application that runs on port 8000, with a dedicated PostgreSQL database.

## Service Type and Instances

**Service Type**: Application Server (Python FastAPI web application)

**Configured Instances**:

- **fastapi-tutorial**: Python FastAPI web application
  - Location/Path: /opt/fastapi-tutorial
  - Port/Socket: 8000
  - Key Config: 
    - Database URL: postgresql://fastapi:fastapi_password@localhost/fastapi_db
    - Project Name: "FastAPI Tutorial"
    - API Version: 1.0.0

- **PostgreSQL Database**: Database backend for the FastAPI application
  - Location/Path: Default PostgreSQL installation path
  - Port/Socket: 5432 (default PostgreSQL port)
  - Key Config:
    - Database: fastapi_db
    - User: fastapi
    - Password: fastapi_password

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
   - Configures PostgreSQL service (enables and starts)
   - Creates database user 'fastapi' with password 'fastapi_password'
   - Creates database 'fastapi_db' owned by 'fastapi' user
   - Grants all privileges on 'fastapi_db' to 'fastapi' user
   - Creates environment configuration file at /opt/fastapi-tutorial/.env with database connection details
   - Creates systemd service file at /etc/systemd/system/fastapi-tutorial.service
   - Reloads systemd daemon when service file changes
   - Enables and starts the fastapi-tutorial service
   - Resources: package (1), directory (1), git (1), execute (3), service (2), file (2)

## Dependencies

**External cookbook dependencies**: None specified in metadata.rb
**System package dependencies**: python3, python3-pip, python3-venv, git, postgresql, postgresql-contrib, libpq-dev
**Service dependencies**: postgresql.service (required for the FastAPI application)

## Checks for the Migration

**Files to verify**:
- /opt/fastapi-tutorial/ (application directory)
- /opt/fastapi-tutorial/venv/ (Python virtual environment)
- /opt/fastapi-tutorial/.env (environment configuration)
- /etc/systemd/system/fastapi-tutorial.service (systemd service file)

**Service endpoints to check**:
- Ports listening: 8000 (FastAPI application), 5432 (PostgreSQL)
- Network interfaces: 0.0.0.0 (FastAPI application listens on all interfaces)

**Templates rendered**:
- No templates are used in this cookbook. Configuration is generated directly in file resources.

## Pre-flight checks:
```bash
# Service status
systemctl status fastapi-tutorial
systemctl status postgresql

# Process verification
ps aux | grep uvicorn
ps aux | grep postgres

# Application health check
curl -I http://localhost:8000/
curl -s http://localhost:8000/docs  # FastAPI auto-generated documentation

# Database connectivity
sudo -u postgres psql -c "SELECT datname FROM pg_database WHERE datname='fastapi_db';"
sudo -u postgres psql -c "SELECT usename FROM pg_user WHERE usename='fastapi';"
PGPASSWORD=fastapi_password psql -h localhost -U fastapi -d fastapi_db -c "SELECT 1;"

# Configuration validation
cat /opt/fastapi-tutorial/.env | grep -E 'PROJECT_NAME|API_VERSION|DATABASE_URL'
cat /etc/systemd/system/fastapi-tutorial.service

# Python environment
ls -la /opt/fastapi-tutorial/venv/bin/
/opt/fastapi-tutorial/venv/bin/python --version
/opt/fastapi-tutorial/venv/bin/pip list

# Git repository
cd /opt/fastapi-tutorial && git remote -v
cd /opt/fastapi-tutorial && git branch

# Logs
journalctl -u fastapi-tutorial -n 100
journalctl -u postgresql -n 50

# Network listening
netstat -tulpn | grep 8000  # FastAPI application
netstat -tulpn | grep 5432  # PostgreSQL
ss -tlnp | grep 8000
ss -tlnp | grep 5432
lsof -i :8000
lsof -i :5432

# Resource usage
ps aux | grep uvicorn | grep -v grep
top -p $(pgrep uvicorn) -n 1
```