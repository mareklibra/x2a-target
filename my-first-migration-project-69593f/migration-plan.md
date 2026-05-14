# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Scope**: 3 Chef cookbooks with dependencies on external cookbooks
**Complexity**: Medium (standard web server patterns with some security hardening)
**Timeline Estimate**: 2-3 weeks for complete migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and self-signed certificates
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw)

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

- `Berksfile`: Defines cookbook dependencies (nginx, memcached, redisio) - will be replaced by Ansible Galaxy requirements
- `solo.json`: Defines the run list and configuration attributes - will be replaced by Ansible inventory variables
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `Vagrantfile`: Defines VM configuration for testing - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible playbook

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in Vagrant testing
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration

### Security Considerations

- **SSL/TLS Management**: 
  - Self-signed certificate generation for development environments
  - Proper file permissions for certificate files (640) and directories (710)
  - Migration approach: Use Ansible crypto modules for certificate generation

- **Firewall Configuration**:
  - UFW configuration for SSH, HTTP, and HTTPS
  - Migration approach: Use Ansible UFW module or firewalld module depending on target OS

- **Fail2ban Integration**:
  - Fail2ban for brute force protection
  - Migration approach: Use Ansible to deploy fail2ban configuration files

- **SSH Hardening**:
  - Disable root login
  - Disable password authentication
  - Migration approach: Use Ansible to configure SSH daemon

- **System Hardening**:
  - Sysctl security configurations
  - Migration approach: Use Ansible sysctl module

- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL password in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - Database connection string in .env file
  - Count: 3 hardcoded credentials identified

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation dynamically generates site configurations based on attributes
  - Mitigation: Use Ansible templates with loops to generate site configurations from variables

- **Self-signed Certificate Generation**:
  - Description: Custom OpenSSL commands for certificate generation
  - Mitigation: Use Ansible's openssl_* modules for certificate management

- **Redis Configuration Patching**:
  - Description: The current implementation uses a ruby_block to modify Redis configuration
  - Mitigation: Use Ansible templates with proper variable substitution instead of post-configuration patching

- **PostgreSQL User and Database Creation**:
  - Description: Uses shell commands to create database and user
  - Mitigation: Use Ansible's postgresql_* modules for proper idempotent database management

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Relatively straightforward to implement in Ansible

2. **cache** (Priority 2)
   - Dependent services that the application will need
   - Moderate complexity due to Redis configuration requirements

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on other infrastructure
   - Higher complexity due to Python environment, Git, and database configuration

### Assumptions

1. The target environment will continue to be either Ubuntu (>= 18.04) or CentOS (>= 7.0)
2. The Vagrant testing environment will be maintained but converted to use Ansible provisioner
3. Self-signed certificates are acceptable for development environments
4. The current security configurations are appropriate and should be maintained
5. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
6. The hardcoded credentials in the current implementation will be replaced with Ansible Vault variables
7. The current directory structure for web content (/var/www/[site]) will be maintained
8. The current PostgreSQL database name and user conventions will be maintained