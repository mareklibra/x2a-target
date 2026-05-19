# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Scope**: 3 Chef cookbooks with external dependencies
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
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef configuration file containing the run list and node attributes
- `solo.rb`: Chef configuration file specifying cookbook paths and log settings
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Vagrant configuration file for the development environment

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be a local development environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role from Galaxy or custom role
- **memcached (~> 6.0)**: Replace with Ansible memcached role from Galaxy
- **redisio (~> 7.2.4)**: Replace with Ansible redis role from Galaxy

### Security Considerations

- **SSL/TLS Configuration**: 
  - Self-signed certificate generation for development
  - Proper certificate and key file permissions
  - Migration approach: Use Ansible's openssl_* modules

- **Firewall Configuration**: 
  - UFW configuration for SSH, HTTP, and HTTPS
  - Migration approach: Use Ansible's ufw module

- **Fail2ban Integration**: 
  - Fail2ban configuration for brute force protection
  - Migration approach: Use Ansible's template module with fail2ban configuration templates

- **SSH Hardening**: 
  - Disable root login and password authentication
  - Migration approach: Use Ansible's lineinfile or template module for sshd_config

- **Vault/secrets management**:
  - Redis password in plain text in recipe (redis_secure_password_123)
  - PostgreSQL password in plain text in recipe (fastapi_password)
  - Migration approach: Use Ansible Vault for storing sensitive credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current setup dynamically generates Nginx site configurations based on node attributes
  - Mitigation: Use Ansible templates with loops to generate site configurations from variables

- **SSL Certificate Management**: 
  - Description: Self-signed certificates are generated for each site
  - Mitigation: Use Ansible's openssl_* modules to generate certificates or integrate with Let's Encrypt

- **Service Orchestration**: 
  - Description: Multiple interdependent services (Nginx, Redis, Memcached, PostgreSQL, FastAPI)
  - Mitigation: Use Ansible handlers and proper dependency ordering

- **PostgreSQL User and Database Creation**: 
  - Description: Current implementation uses inline shell commands
  - Mitigation: Use Ansible's postgresql_* modules for idempotent database management

### Migration Order

1. **cache cookbook** (low risk, standalone services)
   - Memcached and Redis configuration
   - Relatively simple with available Ansible Galaxy roles

2. **nginx-multisite cookbook** (moderate complexity)
   - Base Nginx installation and configuration
   - SSL certificate generation
   - Virtual host configuration
   - Security hardening

3. **fastapi-tutorial cookbook** (high complexity, dependencies)
   - PostgreSQL database setup
   - Python application deployment
   - Environment configuration
   - Systemd service setup

### Assumptions

1. The target environment will continue to be Fedora-based systems
2. Self-signed certificates are acceptable for development environments
3. The same security hardening measures will be maintained
4. The FastAPI application repository will remain available at the specified URL
5. The current network configuration and port mappings will be preserved
6. No changes to the application architecture are required during migration
7. Redis and Memcached configurations will remain similar
8. The migration will not involve changes to the application code itself