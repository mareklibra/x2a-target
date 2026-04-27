# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies, configuration templates, and security settings. Based on the complexity and scope, this migration is estimated to require 3-4 weeks with 1-2 engineers.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, UFW firewall), sysctl security settings

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef configuration file containing the run list and node attributes
- `solo.rb`: Chef configuration file specifying cookbook paths and log settings
- `Vagrantfile`: Defines the development VM using Fedora 42, with port forwarding and network settings
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for local development/testing

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role or memcached module
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or community.general.redis module

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. In Ansible, use the `community.crypto.x509_certificate` module.
- **Firewall Configuration**: UFW firewall rules need to be migrated to Ansible's `community.general.ufw` module.
- **fail2ban Configuration**: Migrate to Ansible's `community.general.fail2ban` module.
- **SSH Hardening**: Current implementation disables root login and password authentication. Use Ansible's `ansible.posix.sshd_config` module.
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook (`fastapi`/`fastapi_password`)
  - Consider using Ansible Vault for these credentials

### Technical Challenges

- **Multi-site Configuration**: The nginx-multisite cookbook dynamically creates site configurations based on node attributes. This pattern needs to be replicated in Ansible using loops and templates.
- **SSL Certificate Generation**: Self-signed certificate generation logic needs to be carefully migrated to maintain the same security properties.
- **Service Dependencies**: The FastAPI application depends on PostgreSQL being configured first. This dependency chain needs to be maintained in Ansible.
- **Configuration Templating**: Multiple configuration templates need to be converted from ERB format to Jinja2 format for Ansible.

### Migration Order

1. **nginx-multisite cookbook** (Priority 1)
   - Core infrastructure component that other services depend on
   - Start with basic Nginx installation and configuration
   - Then add SSL certificate generation
   - Finally add security hardening features

2. **cache cookbook** (Priority 2)
   - Relatively simple cookbook with external dependencies
   - Migrate Memcached configuration first
   - Then migrate Redis configuration with authentication

3. **fastapi-tutorial cookbook** (Priority 3)
   - Application-specific cookbook with database dependencies
   - Migrate PostgreSQL installation and configuration
   - Then migrate Python application deployment
   - Finally migrate systemd service configuration

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as specified in the Vagrantfile).
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or other CA).
3. The hardcoded credentials in the current implementation are for development purposes and will be replaced with more secure solutions in the Ansible implementation.
4. The current firewall and fail2ban configurations are sufficient for security requirements and should be maintained as-is.
5. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available and unchanged.
6. The current Vagrant development workflow should be preserved in the Ansible migration.