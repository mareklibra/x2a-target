# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks to Ansible roles, addressing external dependencies, and ensuring security configurations are properly maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 3 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7 weeks

**Complexity Assessment:** Medium
- Multiple interconnected services
- Security configurations that need careful migration
- External dependencies on community cookbooks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled subdomains, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening with fail2ban and UFW

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file listing cookbook dependencies (nginx, memcached, redisio, ssl_certificate)
- `Policyfile.rb`: Chef Policyfile defining the run list and cookbook dependencies
- `Vagrantfile`: Defines the development VM configuration using Fedora 42
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `solo.json`: Configuration data for Chef Solo, containing site configurations and security settings
- `solo.rb`: Chef Solo configuration file

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) as specified in cookbook metadata files. Development environment uses Fedora 42 as specified in the Vagrantfile.
- **Virtual Machine Technology**: Vagrant with libvirt provider as indicated in the Vagrantfile.
- **Cloud Platform**: Not specified in the repository. The configuration appears to be designed for on-premises or generic cloud VMs.

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible's `geerlingguy.memcached` role or direct package management
- **redisio (~> 7.2.4)**: Replace with Ansible's `geerlingguy.redis` role or direct package management
- **ssl_certificate (~> 2.1)**: Replace with Ansible's `openssl_*` modules for certificate generation

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migrate to Ansible's `openssl_certificate` module or consider integration with Let's Encrypt using `geerlingguy.certbot`.
- **Firewall Configuration**: UFW configuration should be migrated to Ansible's `ufw` module or `firewalld` module depending on the target OS.
- **fail2ban Configuration**: Migrate to Ansible's `fail2ban` module or direct configuration file management.
- **SSH Hardening**: Current configuration disables root login and password authentication. Migrate these settings using Ansible's `lineinfile` or template modules.
- **Redis Password**: The Redis password is hardcoded in the Chef recipe. This should be moved to Ansible Vault for secure storage.
- **PostgreSQL Credentials**: Database credentials for the FastAPI application should be stored in Ansible Vault.

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of site configurations based on attributes will need to be replicated using Ansible's templating system.
- **SSL Certificate Generation**: Self-signed certificate generation logic needs to be carefully migrated to maintain security.
- **System Hardening**: Security configurations in the `security.rb` recipe need careful migration to ensure no security measures are missed.
- **Service Dependencies**: Ensuring proper ordering of service deployments (PostgreSQL before FastAPI, etc.) will require careful planning of Ansible role dependencies.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Implement site configuration templates
   - Add security hardening features

2. **cache** (low complexity, standalone service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - Implement PostgreSQL installation and configuration
   - Set up Python environment and application deployment
   - Configure systemd service

### Assumptions

1. The target environment will continue to support the same operating systems (Ubuntu 18.04+ or CentOS 7+).
2. Self-signed certificates are acceptable for the migrated solution, or a migration to Let's Encrypt will be part of the scope.
3. The FastAPI application repository at `https://github.com/dibanez/fastapi_tutorial.git` will remain available.
4. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment.
5. Redis and Memcached configurations can be migrated without changes to their core functionality.
6. The Vagrant development environment will be replaced with an Ansible-compatible alternative.
7. No custom Chef resources or libraries are used beyond what is visible in the repository structure.