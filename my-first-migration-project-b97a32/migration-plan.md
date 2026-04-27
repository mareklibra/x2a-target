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
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file listing both local and external cookbook dependencies
- `solo.json`: Chef configuration file containing the run list and node attributes
- `solo.rb`: Chef configuration file specifying cookbook paths and log settings
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Vagrant configuration file for the development environment

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be a local development environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or dedicated tasks for Redis installation and configuration

### Security Considerations

- **SSL/TLS Configuration**: 
  - Self-signed certificate generation for development
  - Proper file permissions for private keys
  - Migration approach: Use Ansible's openssl_* modules or community.crypto collection

- **Firewall Configuration**: 
  - UFW rules for SSH, HTTP, and HTTPS
  - Migration approach: Use Ansible's ufw module

- **Fail2ban Integration**: 
  - Custom jail configuration
  - Migration approach: Use Ansible's template module for fail2ban configuration

- **System Hardening**:
  - Sysctl security parameters
  - SSH hardening (root login disabled, password authentication disabled)
  - Migration approach: Use Ansible's sysctl and lineinfile modules

- **Vault/secrets management**:
  - Redis password in plaintext in the cache cookbook
  - PostgreSQL credentials in plaintext in the fastapi-tutorial cookbook
  - Migration approach: Use Ansible Vault for sensitive credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: The current implementation dynamically generates site configurations based on node attributes. This will need to be replicated using Ansible's template module and variables.
  - Mitigation: Create a structured variable in Ansible inventory or group_vars to define sites and their properties

- **SSL Certificate Generation**: The current implementation generates self-signed certificates using inline shell commands.
  - Mitigation: Use Ansible's openssl_certificate module for more maintainable certificate generation

- **Redis Configuration Hack**: The cache cookbook includes a ruby_block to modify Redis configuration files after they're created.
  - Mitigation: Create a proper Redis configuration template in Ansible rather than modifying files after creation

- **PostgreSQL User and Database Creation**: The current implementation uses inline shell commands to create database users and permissions.
  - Mitigation: Use Ansible's postgresql_* modules for more idempotent database management

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Relatively self-contained with clear configuration templates

2. **cache** (Priority 2)
   - Depends on external cookbooks that need to be replaced with Ansible roles
   - Contains sensitive information that needs to be secured with Ansible Vault

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on properly configured infrastructure
   - Involves multiple components (Python, PostgreSQL, systemd)

### Assumptions

1. The target environment will continue to be Fedora-based systems
2. Self-signed certificates are acceptable for development environments
3. The same security hardening measures should be maintained in the Ansible implementation
4. The FastAPI application repository will remain available at the specified URL
5. The current directory structure in the target environment (/opt/server/*, /var/www/*) should be preserved
6. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment
7. Redis and Memcached configurations should maintain the same performance characteristics
8. The Nginx virtual hosts configuration should maintain the same structure and security headers