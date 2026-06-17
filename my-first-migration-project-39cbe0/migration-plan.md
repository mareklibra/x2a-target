# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained. Based on the repository analysis, this is a medium complexity migration that should take approximately 3-4 weeks to complete with a small team (2-3 engineers).

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

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints.
- `solo.json`: Chef Solo configuration file containing the run list and node attributes.
- `solo.rb`: Chef Solo configuration file specifying file paths and log settings.
- `Vagrantfile`: Defines a Vagrant VM configuration using Fedora 42 with port forwarding and resource allocation.
- `vagrant-provision.sh`: Shell script that installs Chef and runs the Chef Solo provisioning process in the Vagrant VM.

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) as specified in cookbook metadata, but the Vagrantfile uses Fedora 42.
- **Virtual Machine Technology**: Vagrant with libvirt provider as specified in the Vagrantfile.
- **Cloud Platform**: Not specified. The configuration appears to be designed for local development/testing environments.

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role (e.g., geerlingguy.nginx)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., geerlingguy.memcached)
- **redisio (~> 7.2.4)**: Replace with Ansible redis role (e.g., geerlingguy.redis)

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW with specific rules for SSH, HTTP, and HTTPS. Ansible migration should use the `ansible.posix.firewalld` or `community.general.ufw` modules.
- **Fail2ban Setup**: The Chef cookbook configures fail2ban for brute force protection. Ansible migration should use the `community.general.fail2ban` module.
- **SSH Hardening**: The Chef cookbook disables root login and password authentication. Ansible migration should use the `ansible.posix.sshd` module.
- **System Hardening**: The Chef cookbook applies sysctl security settings. Ansible migration should use the `ansible.posix.sysctl` module.
- **Vault/secrets management**: 
  - Redis password is hardcoded in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook (`fastapi`/`fastapi_password`)
  - SSL certificates are generated on the fly with self-signed certificates
  - Total credentials detected: 2 sets of database credentials, 1 Redis password

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations based on node attributes. Ansible will need to use templates and loops to achieve the same functionality.
- **SSL Certificate Generation**: The Chef cookbook generates self-signed SSL certificates for each site. Ansible will need to use the `community.crypto` collection for certificate management.
- **Redis Configuration Hack**: The Chef cookbook includes a Ruby block to modify Redis configuration files after they're created. Ansible will need to use templates or lineinfile modules to achieve the same result.
- **PostgreSQL User/Database Creation**: The Chef cookbook uses shell commands to create PostgreSQL users and databases. Ansible should use the `community.postgresql` collection for better idempotence.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Add security hardening (fail2ban, ufw, sysctl)
   - Add multi-site configuration

2. **cache** (low complexity, standalone service)
   - Implement Memcached configuration
   - Implement Redis configuration with authentication

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - Implement PostgreSQL installation and configuration
   - Implement Python environment setup
   - Implement application deployment from Git
   - Implement systemd service configuration

### Assumptions

1. The target environment will continue to be Fedora-based as specified in the Vagrantfile, despite the cookbooks supporting Ubuntu and CentOS.
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or other certificate authority).
3. The hardcoded credentials in the Chef cookbooks are for development purposes and will be replaced with Ansible Vault in the production migration.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git is accessible and will remain available.
5. The current Chef setup is functional and represents the desired end state for the Ansible migration.
6. The Vagrant development workflow should be preserved in the Ansible migration.