# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated complexity is medium, with an estimated timeline of 3-4 weeks for a complete migration.

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

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef configuration file containing the run list and node attributes
- `solo.rb`: Chef configuration file specifying cookbook paths and log settings
- `Vagrantfile`: Defines the development VM configuration using Fedora 42
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora (based on Vagrantfile specifying "generic/fedora42"), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or local development

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role `geerlingguy.nginx` or create a custom Nginx role
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or DavidWittman.redis

### Security Considerations

- **SSL Certificate Management**: 
  - Migration approach: Use Ansible's `openssl_*` modules for certificate generation or consider integrating with Ansible Vault for certificate storage
  - Current implementation uses self-signed certificates generated on the fly

- **Firewall Configuration**: 
  - Migration approach: Use Ansible's `ufw` module to replace the current Chef UFW implementation

- **Fail2ban Integration**: 
  - Migration approach: Create an Ansible role for fail2ban configuration using templates

- **SSH Hardening**: 
  - Migration approach: Use Ansible's `lineinfile` module or the `ansible.posix.sshd` module to configure SSH security settings

- **Vault/secrets management**:
  - Redis password in plaintext in the cache cookbook (redis_secure_password_123)
  - PostgreSQL credentials in plaintext in the fastapi-tutorial cookbook (fastapi/fastapi_password)
  - Migration approach: Move all credentials to Ansible Vault

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation dynamically generates Nginx site configurations based on node attributes
  - Mitigation strategy: Create Ansible templates with Jinja2 loops to generate site configurations from variables

- **Redis Configuration Patching**: 
  - Description: The Chef cookbook uses a ruby_block to modify Redis configuration files after they're created
  - Mitigation strategy: Create a custom Redis configuration template in Ansible that properly handles the configuration from the start

- **Service Orchestration**: 
  - Description: The current implementation has specific service ordering (e.g., PostgreSQL before FastAPI)
  - Mitigation strategy: Use Ansible handlers and the `notify` mechanism to ensure proper service restart ordering

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for web services)
   - Create base Nginx role
   - Implement SSL certificate generation
   - Configure security headers and firewall rules

2. **cache** (low complexity, standalone services)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, application deployment)
   - Implement PostgreSQL database setup
   - Configure Python environment and application deployment
   - Set up systemd service

### Assumptions

1. The target environment will continue to be Fedora-based systems, with potential support for Ubuntu and CentOS as indicated in the cookbook metadata.
2. Self-signed certificates are acceptable for the migration; no integration with Let's Encrypt or other certificate authorities is required.
3. The current security configurations (fail2ban, UFW, SSH hardening) should be maintained in the Ansible implementation.
4. The FastAPI application source will continue to be pulled from the same Git repository.
5. The Redis and PostgreSQL passwords currently hardcoded in the recipes will be moved to a secure storage mechanism like Ansible Vault.
6. The Vagrant development environment should be preserved but updated to use Ansible provisioning instead of Chef.
7. No changes to the application architecture or deployment strategy are required as part of the migration.