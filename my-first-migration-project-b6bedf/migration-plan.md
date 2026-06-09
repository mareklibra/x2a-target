# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated timeline for this migration is 3-4 weeks, with moderate complexity due to the security configurations and multiple service integrations.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw, sysctl)

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with file paths and log settings
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Vagrant configuration file for development environment using Fedora 42

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration

### Security Considerations

- **Firewall Configuration**: The migration must preserve the UFW firewall rules for SSH, HTTP, and HTTPS
- **Fail2ban Integration**: Fail2ban configuration must be migrated to protect against brute force attacks
- **SSH Hardening**: SSH security configurations (disabling root login, password authentication) must be preserved
- **System Hardening**: sysctl security configurations must be migrated
- **Vault/secrets management**:
  - Redis authentication password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL database credentials in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - SSL certificates and private keys managed in nginx-multisite cookbook

### Technical Challenges

- **SSL Certificate Management**: The current implementation generates self-signed certificates. The Ansible migration should maintain this functionality while allowing for future integration with Let's Encrypt or other certificate providers.
- **Multi-site Configuration**: The nginx-multisite cookbook dynamically creates site configurations based on node attributes. This dynamic behavior needs to be replicated in Ansible.
- **Service Dependencies**: The FastAPI application depends on PostgreSQL, and the nginx configuration depends on the SSL certificates. These dependencies need to be properly managed in the Ansible playbooks.
- **Redis Configuration Hack**: The cache cookbook contains a "hack" to fix Redis configuration files. This needs to be properly addressed in the Ansible migration.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for web services)
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Implement security hardening (fail2ban, ufw, sysctl)
   - Configure virtual hosts

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Address the Redis configuration hack

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - Set up PostgreSQL database and user
   - Deploy FastAPI application from Git
   - Configure Python environment and dependencies
   - Set up systemd service

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions (Ubuntu 18.04+, CentOS 7+).
2. Self-signed SSL certificates are acceptable for the migrated solution, but the implementation should be flexible enough to allow for proper certificates in production.
3. The hardcoded credentials in the Chef recipes (Redis password, PostgreSQL password) will be replaced with Ansible Vault variables for improved security.
4. The current directory structure in the target system (/opt/fastapi-tutorial, /etc/ssl/certs, etc.) will be maintained for compatibility.
5. The Vagrant development environment will be replaced with an equivalent Ansible-based development workflow.
6. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment and should be maintained.
7. The nginx-multisite cookbook's dynamic site configuration based on node attributes will be replicated using Ansible variables and templates.
8. The Redis configuration "hack" in the cache cookbook is necessary and should be implemented in a cleaner way in Ansible.