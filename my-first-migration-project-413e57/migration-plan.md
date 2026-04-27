# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Scope**: 3 Chef cookbooks with dependencies on external cookbooks
**Complexity**: Medium
**Estimated Timeline**: 3-4 weeks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and firewall configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security headers, fail2ban integration, UFW firewall configuration

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies including nginx (~> 12.0), memcached (~> 6.0), and redisio (~> 7.2.4)
- `solo.json`: Defines the run list and configuration attributes for nginx sites and security settings
- `solo.rb`: Chef Solo configuration file specifying cookbook paths and log settings
- `Vagrantfile`: Defines a Fedora 42 VM for development/testing with port forwarding and networking
- `vagrant-provision.sh`: Bash script to provision the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora (based on Vagrantfile using "generic/fedora42"), with support for Ubuntu (>= 18.04) and CentOS (>= 7.0) mentioned in cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration
- **PostgreSQL**: Replace with Ansible postgresql_* modules or community.postgresql collection

### Security Considerations

- **SSL/TLS Configuration**: 
  - Self-signed certificate generation needs to be migrated to Ansible's openssl_* modules
  - Maintain strong cipher configurations and security headers

- **Firewall Configuration**: 
  - UFW configuration should be migrated to Ansible's ufw module
  - Maintain existing allowed ports (SSH, HTTP, HTTPS)

- **Fail2ban Integration**:
  - Migrate fail2ban configuration to Ansible's fail2ban_jail module

- **SSH Hardening**:
  - Maintain root login disable and password authentication disable settings

- **Vault/secrets management**:
  - Redis password ("redis_secure_password_123") in cache/recipes/default.rb
  - PostgreSQL user password ("fastapi_password") in fastapi-tutorial/recipes/default.rb
  - Database connection string in .env file
  - Total credentials detected: 3 (should be migrated to Ansible Vault)

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Challenge: Dynamically generating multiple virtual host configurations with SSL
  - Mitigation: Use Ansible templates with loops to generate site configurations similar to the Chef approach

- **Service Orchestration**: 
  - Challenge: Ensuring services start in the correct order (PostgreSQL before FastAPI app)
  - Mitigation: Use Ansible handlers and proper dependency management with meta dependencies

- **SSL Certificate Management**:
  - Challenge: Generating and managing self-signed certificates
  - Mitigation: Use Ansible's openssl_* modules with proper idempotency checks

- **Python Application Deployment**:
  - Challenge: Managing Python virtual environments and dependencies
  - Mitigation: Use Ansible's pip module with virtualenv parameter

### Migration Order

1. **nginx-multisite cookbook** (Priority 1)
   - Base infrastructure component that other services depend on
   - Relatively self-contained with clear configuration templates

2. **cache cookbook** (Priority 2)
   - Depends on external cookbooks (memcached, redisio)
   - Requires careful handling of authentication settings

3. **fastapi-tutorial cookbook** (Priority 3)
   - Most complex with database, application code, and service management
   - Depends on proper functioning of web and database services

### Assumptions

1. The target environment will continue to be Fedora-based, with potential support for Ubuntu and CentOS as mentioned in cookbook metadata
2. Self-signed certificates are acceptable for the migrated environment (production would likely require proper certificates)
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
4. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment
5. Redis and PostgreSQL passwords in the code are development passwords that will be replaced with proper secrets management in production
6. The Vagrant development environment will be maintained, but Chef-specific provisioning will be replaced with Ansible