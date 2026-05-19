# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Scope**: 3 Chef cookbooks with 3 external dependencies
**Complexity**: Medium
**Estimated Timeline**: 3-4 weeks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and self-signed certificate generation
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL/TLS setup, security hardening (fail2ban, ufw, sysctl)

- **renamed-fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

- **something-new**:
    - Description: A new module
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: none

### Infrastructure Files

- `Berksfile`: Chef dependency manager file listing cookbook dependencies (nginx, memcached, redisio)
- `solo.json`: Chef Solo configuration with run list and node attributes
- `solo.rb`: Chef Solo configuration file with cookbook paths and log settings
- `Vagrantfile`: Defines development VM using Fedora 42 with port forwarding and networking
- `vagrant-provision.sh`: Bash script to provision the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in Vagrant
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or collection (e.g., `ansible.posix.nginx`)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., `geerlingguy.memcached`)
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (e.g., `geerlingguy.redis`)

### Security Considerations

- **SSL/TLS Management**: 
  - Migration approach: Use Ansible's `openssl_*` modules for certificate generation
  - Consider integrating with Ansible Vault for certificate storage

- **Firewall Configuration (UFW)**:
  - Migration approach: Use Ansible's `ufw` module to configure firewall rules

- **Fail2ban Configuration**:
  - Migration approach: Use Ansible templates to configure fail2ban jails

- **SSH Hardening**:
  - Migration approach: Use Ansible's `lineinfile` or templates to configure SSH daemon

- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL credentials in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - Consider using Ansible Vault to secure these credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation uses Chef attributes and templates to generate multiple virtual host configurations
  - Mitigation: Create Ansible templates with loops to generate similar configurations from variables

- **Self-signed Certificate Generation**:
  - Description: Custom SSL certificate generation for each virtual host
  - Mitigation: Use Ansible's `openssl_certificate` module with appropriate parameters

- **Redis Configuration Patching**:
  - Description: The Chef cookbook uses a ruby_block to modify Redis configuration files after they're created
  - Mitigation: Create a custom Redis configuration template in Ansible that doesn't require post-processing

- **PostgreSQL User and Database Creation**:
  - Description: Uses shell commands via execute resources
  - Mitigation: Use Ansible's `postgresql_*` modules for idempotent database management

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Add security hardening (fail2ban, ufw)
   - Add virtual host configuration

2. **cache** (low complexity, standalone service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, application deployment)
   - Implement PostgreSQL database setup
   - Implement Python environment and application deployment
   - Configure systemd service

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS/RHEL-based distributions
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or other CA)
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
4. The current hardcoded credentials will be replaced with more secure solutions using Ansible Vault
5. The Vagrant development environment will be maintained but converted to use Ansible provisioner
6. No changes to the application architecture or deployment model are required
7. The current security configurations (fail2ban, ufw, SSH hardening) are sufficient and should be maintained
