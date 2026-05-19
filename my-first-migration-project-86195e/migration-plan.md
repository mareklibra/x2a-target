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
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw, sysctl)

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

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible requirements.yml
- `solo.json`: Chef node attributes and run list - will be replaced by Ansible inventory variables
- `solo.rb`: Chef configuration - will be replaced by ansible.cfg
- `Vagrantfile`: Development environment configuration - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible playbook

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or local development environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks

### Security Considerations

- **SSL Certificate Management**: 
  - Self-signed certificates are generated for development
  - Migration should maintain proper certificate permissions (640) and ownership (root:ssl-cert)
  - Consider using Ansible's community.crypto.openssl_* modules

- **Firewall Configuration**: 
  - UFW firewall rules need to be migrated using ansible.posix.ufw module
  - Default deny policy with specific allows for SSH, HTTP, HTTPS

- **Fail2ban Configuration**:
  - Migrate fail2ban configuration using Ansible templates
  - Ensure service is enabled and running

- **SSH Hardening**:
  - Disable root login
  - Disable password authentication
  - Use ansible.posix.sysctl for kernel parameter hardening

- **Vault/secrets management**:
  - Redis password in cache cookbook (plaintext "redis_secure_password_123")
  - PostgreSQL password in fastapi-tutorial cookbook (plaintext "fastapi_password")
  - Consider using Ansible Vault for these credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - The dynamic generation of multiple virtual hosts based on node attributes will need to be replicated using Ansible loops and templates
  - Solution: Use Ansible with_items/loop constructs with templates

- **Redis Configuration Hack**: 
  - The Chef cookbook includes a ruby_block to modify Redis configuration
  - Solution: Use Ansible lineinfile or template module with proper configuration

- **PostgreSQL User/Database Creation**:
  - Current implementation uses shell commands via execute resource
  - Solution: Use community.postgresql modules for cleaner implementation

- **Service Dependencies**:
  - FastAPI service depends on PostgreSQL
  - Solution: Use Ansible handlers and meta dependencies to ensure proper ordering

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Add security hardening components
   - Add virtual host configuration

2. **cache** (Priority 2)
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Ensure proper service management

3. **fastapi-tutorial** (Priority 3)
   - Set up PostgreSQL database
   - Deploy Python application
   - Configure environment variables
   - Set up systemd service

### Assumptions

1. The target environment will continue to be Fedora-based systems (Fedora 42 as specified in Vagrantfile)
2. Self-signed certificates are acceptable for development (production would likely use Let's Encrypt or other CA)
3. The security requirements will remain the same (fail2ban, ufw, SSH hardening)
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. The current plaintext passwords in the Chef recipes are for development only and will be replaced with Ansible Vault encrypted values
6. The Vagrant development environment will be maintained but converted to use Ansible provisioner
7. No custom Chef resources are being used that would require special handling in Ansible
8. The current multi-site configuration pattern will be maintained in the Ansible implementation