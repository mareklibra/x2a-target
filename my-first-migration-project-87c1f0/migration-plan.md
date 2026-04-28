# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Scope**: 3 Chef cookbooks with external dependencies
**Complexity**: Medium
**Estimated Timeline**: 3-4 weeks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and firewall configuration
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
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file listing cookbook dependencies (nginx, memcached, redisio)
- `solo.json`: Chef configuration file defining the run list and node attributes
- `solo.rb`: Chef configuration file defining paths and log settings
- `Vagrantfile`: Defines a Fedora 42 VM for local development with port forwarding
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile) with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role `geerlingguy.nginx` or create a custom Nginx role
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or DavidWittman.redis

### Security Considerations

- **SSL Certificate Management**: 
  - Migration approach: Use Ansible's `openssl_*` modules for certificate generation or consider integrating with Let's Encrypt via `geerlingguy.certbot`
  - Current implementation uses self-signed certificates

- **Firewall Configuration**: 
  - Migration approach: Use Ansible's `ufw` module to replicate the current UFW configuration

- **Fail2ban Integration**: 
  - Migration approach: Create an Ansible role for fail2ban configuration using templates

- **SSH Hardening**: 
  - Migration approach: Use Ansible's `lineinfile` module or the `devsec.hardening.ssh_hardening` role

- **Vault/secrets management**:
  - Redis password in plaintext in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL password in plaintext in the fastapi-tutorial cookbook (`fastapi_password`)
  - Consider using Ansible Vault for these credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation dynamically generates site configurations based on node attributes
  - Mitigation: Create Ansible templates with Jinja2 loops to replicate the dynamic site generation

- **Redis Configuration Patching**: 
  - Description: The current implementation includes a Ruby block to modify Redis configuration files after installation
  - Mitigation: Use Ansible's `lineinfile` or `replace` modules to make the necessary modifications to Redis configuration

- **PostgreSQL User and Database Creation**: 
  - Description: The current implementation uses shell commands to create PostgreSQL users and databases
  - Mitigation: Use Ansible's `postgresql_*` modules for more idiomatic database management

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Supporting service with external dependencies
   - Moderate complexity due to Redis configuration patching

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on the infrastructure being in place
   - Involves multiple components (Python, Git, PostgreSQL)

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions
2. Self-signed certificates are acceptable for development, but production may require proper certificates
3. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. The current Redis and PostgreSQL passwords are development passwords and will be replaced with secure passwords in production
6. The current directory structure in `/opt` and `/var/www` will be maintained in the target environment
7. The Nginx site configurations (test.cluster.local, ci.cluster.local, status.cluster.local) will remain the same
8. The current port mappings (80, 443, 8000) will be maintained