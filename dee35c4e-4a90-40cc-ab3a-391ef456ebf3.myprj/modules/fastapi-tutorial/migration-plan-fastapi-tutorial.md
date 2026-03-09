# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook deploys a FastAPI Python web application with a PostgreSQL database backend. It sets up a single instance of the application running on port 8000, with key features including Python virtual environment, Git repository deployment, and systemd service management.

## Service Type and Instances

**Service Type**: Web Server (FastAPI Python Application)

**Configured Instances**:
- **fastapi-tutorial**: Python FastAPI web application
  - Location/Path: /opt/fastapi-tutorial
  - Port/Socket: 8000
  - Key Config: 
    - Database: PostgreSQL (fastapi_db)
    - Database User: fastapi
    - Environment Variables: PROJECT_NAME, API_VERSION, DATABASE_URL

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
   - Clones FastAPI tutorial repository from GitHub (https://github.com/dibanez/fastapi_tutorial.git, branch: main)
   - Creates Python virtual environment at /opt/fastapi-tutorial/venv
   - Installs Python dependencies from requirements.txt
   - Configures PostgreSQL service (enable and start)
   - Creates database user 'fastapi' with password 'fastapi_password'
   - Creates database 'fastapi_db' owned by 'fastapi' user
   - Creates environment configuration file (.env) with PROJECT_NAME, API_VERSION, and DATABASE_URL
   - Creates systemd service file for fastapi-tutorial
   - Reloads systemd daemon when service file changes
   - Enables and starts the fastapi-tutorial service
   - Resources: package (1), directory (1), git (1), execute (3), service (2), file (2)

## Dependencies

**External cookbook dependencies**: None specified in metadata.rb
**System package dependencies**: python3, python3-pip, python3-venv, git, postgresql, postgresql-contrib, libpq-dev
**Service dependencies**: postgresql.service (required before fastapi-tutorial.service)

## Checks for the Migration

**Files to verify**:
- /opt/fastapi-tutorial (application directory)
- /opt/fastapi-tutorial/venv (Python virtual environment)
- /opt/fastapi-tutorial/.env (Environment configuration)
- /etc/systemd/system/fastapi-tutorial.service (Systemd service file)

**Service endpoints to check**:
- Ports listening: 8000 (FastAPI application)
- Network interfaces: 0.0.0.0 (listens on all interfaces)

**Templates rendered**:
- No templates used, but two files are created with inline content:
  - /opt/fastapi-tutorial/.env
  - /etc/systemd/system/fastapi-tutorial.service

## Pre-flight checks:
```bash
# Service status
systemctl status fastapi-tutorial
systemctl status postgresql

# Process verification
ps aux | grep uvicorn
ps aux | grep fastapi

# Application health check
curl -I http://localhost:8000/
curl -s http://localhost:8000/docs  # FastAPI auto-generated docs

# Database connectivity
sudo -u postgres psql -c "\l" | grep fastapi_db
sudo -u postgres psql -c "\du" | grep fastapi
psql -U fastapi -h localhost -d fastapi_db -c "SELECT 1;"

# Configuration validation
cat /opt/fastapi-tutorial/.env
cat /etc/systemd/system/fastapi-tutorial.service
grep ExecStart /etc/systemd/system/fastapi-tutorial.service

# Python environment
ls -la /opt/fastapi-tutorial/venv/bin/
/opt/fastapi-tutorial/venv/bin/python --version
/opt/fastapi-tutorial/venv/bin/pip list | grep fastapi
/opt/fastapi-tutorial/venv/bin/pip list | grep uvicorn

# Git repository
cd /opt/fastapi-tutorial && git remote -v
cd /opt/fastapi-tutorial && git branch

# Logs
journalctl -u fastapi-tutorial -n 100 --no-pager
journalctl -u postgresql -n 50 --no-pager

# Network listening
netstat -tulpn | grep 8000
ss -tulpn | grep 8000
lsof -i :8000

# Resource usage
ps aux | grep uvicorn | grep -v grep
top -b -n 1 | grep uvicorn
```