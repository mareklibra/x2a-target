# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site web server with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated timeline for migration is 2-3 weeks, with moderate complexity due to the security configurations and multi-site setup.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and self-signed certificates
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
- `Vagrantfile`: Defines the development VM configuration using Fedora 42 with port forwarding and networking.
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef Solo.

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be a local development environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW. Migration should use Ansible's `ufw` module.
- **Fail2ban Setup**: The Chef cookbook configures fail2ban. Migration should use Ansible's `fail2ban` module or template tasks.
- **SSH Hardening**: The Chef cookbook disables root login and password authentication. Migration should use Ansible's `lineinfile` or templates for SSH configuration.
- **SSL Certificate Management**: Self-signed certificates are generated for each site. Migration should use Ansible's `openssl_*` modules.
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook (`fastapi`/`fastapi_password`)
  - No Chef Vault or encrypted data bags are used in the current implementation

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations based on node attributes. Ansible implementation will need to use loops with templates.
- **SSL Certificate Generation**: Self-signed certificates are generated for each site. Ansible implementation will need to use the `openssl_*` modules.
- **Service Dependencies**: The FastAPI application depends on PostgreSQL. Ansible implementation will need to ensure proper service ordering.
- **Configuration File Modifications**: The Redis configuration file is modified with a Ruby block. Ansible implementation will need to use `lineinfile` or templates.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Create base Nginx role with security hardening
   - Implement multi-site configuration with SSL
   
2. **cache** (low complexity, standalone service)
   - Implement Memcached configuration
   - Implement Redis with authentication
   
3. **fastapi-tutorial** (high complexity, application deployment)
   - Implement PostgreSQL database setup
   - Implement Python application deployment
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora-based systems (the current configuration uses Fedora 42).
2. Self-signed certificates are acceptable for the migrated solution (production environments might require Let's Encrypt or other certificate authorities).
3. The hardcoded credentials in the Chef recipes will be replaced with Ansible Vault or another secure credential management solution.
4. The FastAPI application source will continue to be available at the specified Git repository.
5. The Vagrant development environment will be maintained, but Chef Solo will be replaced with Ansible.
6. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
7. The current service configurations (Nginx, Memcached, Redis, PostgreSQL) are appropriate for the target environment.