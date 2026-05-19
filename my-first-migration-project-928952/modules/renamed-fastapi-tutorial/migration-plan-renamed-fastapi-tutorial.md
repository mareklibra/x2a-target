---
source-path: cookbooks/fastapi-tutorial
---

# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook installs and configures a FastAPI Python web application with PostgreSQL database backend. It sets up a single instance of the application running on port 8000, with a dedicated database and systemd service.

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
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/fastapi-tutorial/recipes/default.rb`):
   - Installs required system packages: python3, python3-pip, python3-venv, git, postgresql, postgresql-contrib, libpq-dev
   - Creates application directory: /opt/fastapi-tutorial
   - Clones FastAPI tutorial repository from https://github.com/dibanez/fastapi_tutorial.git (branch: main)
   - Creates Python virtual environment at /opt/fastapi-tutorial/venv
   - Installs Python dependencies from requirements.txt
   - Enables and starts PostgreSQL service
   - Creates PostgreSQL database (fastapi_db) and user (fastapi) with password
   - Creates environment file (.env) with application configuration
   - Creates systemd service file for the FastAPI application
   - Enables and starts the FastAPI application service
   - Resources: package (1), directory (1), git (1), execute (3), service (2), file (2)

## Dependencies

**External cookbook dependencies**: None specified in metadata.rb
**System package dependencies**: python3, python3-pip, python3-venv, git, postgresql, postgresql-contrib, libpq-dev
**Service dependencies**: postgresql.service (required for the FastAPI application)

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
- **Usage context**: PostgreSQL database user password for the 'fastapi' user

### Database Connection String

- **Variable(s)**: 'DATABASE_URL'
- **Source file(s)**: cookbooks/fastapi-tutorial/recipes/default.rb
- **Current storage**: hardcoded in .env file
- **Usage context**: Database connection string for the FastAPI application to connect to PostgreSQL

## Checks for the Migration

**Files to verify**:
- /opt/fastapi-tutorial (application directory)
- /opt/fastapi-tutorial/venv (Python virtual environment)
- /opt/fastapi-tutorial/.env (environment configuration)
- /etc/systemd/system/fastapi-tutorial.service (systemd service file)

**Service endpoints to check**:
- Ports listening: 8000 (FastAPI application), 5432 (PostgreSQL)
- Network interfaces: 0.0.0.0 (FastAPI application listens on all interfaces)

**Templates rendered**:
- No templates used, direct file content creation

## Pre-flight checks:
```bash
# Service status
systemctl status fastapi-tutorial
systemctl status postgresql

# Process verification
ps aux | grep uvicorn
ps aux | grep postgres

# Application health
curl -I http://localhost:8000/
curl -s http://localhost:8000/docs  # FastAPI auto-generated docs

# Database connectivity
sudo -u postgres psql -c "\l" | grep fastapi_db
sudo -u postgres psql -c "\du" | grep fastapi
PGPASSWORD=fastapi_password psql -h localhost -U fastapi -d fastapi_db -c "SELECT 1;"

# Configuration validation
cat /opt/fastapi-tutorial/.env
cat /etc/systemd/system/fastapi-tutorial.service
ls -la /opt/fastapi-tutorial/venv/bin/  # Check if virtual environment is properly set up

# Git repository check
cd /opt/fastapi-tutorial && git remote -v
cd /opt/fastapi-tutorial && git branch

# Python dependencies
/opt/fastapi-tutorial/venv/bin/pip list

# Logs
journalctl -u fastapi-tutorial -n 100
journalctl -u postgresql -n 50

# Network listening
netstat -tulpn | grep 8000  # FastAPI application
netstat -tulpn | grep 5432  # PostgreSQL
ss -tlnp | grep 8000
lsof -i :8000

# Resource usage
ps aux | grep uvicorn | grep -v grep
top -p $(pgrep uvicorn) -n 1
```