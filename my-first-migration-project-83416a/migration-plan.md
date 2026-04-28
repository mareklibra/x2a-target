# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup with three primary cookbooks: nginx-multisite, cache, and fastapi-tutorial. The migration to Ansible will involve converting Chef recipes, templates, and attributes to Ansible roles, playbooks, and variables. The estimated timeline for this migration is 3-4 weeks, with moderate complexity due to the interdependencies between services and security configurations.

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
    - Key Features: Python virtual environment, Git repository deployment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints.
- `solo.json`: Chef configuration file defining the run list and node attributes. Contains site configurations and security settings.
- `solo.rb`: Chef configuration file specifying cookbook paths and logging settings.
- `Vagrantfile`: Defines a Vagrant VM configuration for development/testing with Fedora 42.
- `vagrant-provision.sh`: Shell script that installs Chef and runs the cookbooks in the Vagrant environment.

### Target Details

- **Operating System**: Fedora (based on Vagrantfile specifying "generic/fedora42"), with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or collection (e.g., `ansible.posix.nginx`)
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role or direct package installation
- **PostgreSQL**: Replace with Ansible PostgreSQL role (e.g., `community.postgresql`)

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW. Migration should use Ansible's `ansible.posix.firewalld` or `community.general.ufw` modules.
- **Fail2ban Setup**: The Chef cookbook configures fail2ban. Migration should use Ansible's fail2ban modules or templates.
- **SSH Hardening**: The Chef cookbook disables root login and password authentication. Migration should use Ansible's `ansible.posix.sshd` module.
- **SSL Certificate Management**: The Chef cookbook generates self-signed certificates. Migration should use Ansible's `community.crypto` collection.
- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL database credentials in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - Environment variables in .env file for FastAPI application
  - Consider using Ansible Vault for these secrets

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates multiple virtual hosts. Ansible implementation will need to use loops and templates to achieve the same functionality.
- **SSL Certificate Generation**: The Chef cookbook generates self-signed certificates. Ansible implementation will need to use the `community.crypto` collection for certificate generation.
- **Service Interdependencies**: The FastAPI application depends on PostgreSQL, and Nginx depends on the FastAPI service. Ansible implementation will need to handle these dependencies correctly.
- **Redis Configuration Hack**: The Chef cookbook includes a ruby_block to modify Redis configuration. Ansible implementation will need to use templates or lineinfile modules to achieve the same result.

### Migration Order

1. **cache cookbook** (low complexity, foundational service)
   - Implement Memcached and Redis configurations
   - Address Redis authentication

2. **fastapi-tutorial cookbook** (moderate complexity)
   - Implement PostgreSQL database setup
   - Deploy FastAPI application from Git
   - Configure systemd service

3. **nginx-multisite cookbook** (high complexity)
   - Implement security configurations (fail2ban, ufw, sysctl)
   - Generate SSL certificates
   - Configure multiple virtual hosts

### Assumptions

1. The target environment will continue to be Fedora-based, with support for Ubuntu and CentOS.
2. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
3. Self-signed certificates are acceptable for the target environment (no Let's Encrypt or commercial certificates required).
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
5. The current Redis and Memcached configurations are appropriate for the target environment.
6. The current PostgreSQL database configuration is appropriate for the target environment.
7. The current Nginx virtual host configurations are appropriate for the target environment.
8. No additional monitoring or logging solutions are required beyond what's currently implemented.
9. The migration will not involve changes to the application code or database schema.
10. The current hardcoded credentials will be replaced with Ansible Vault secrets.