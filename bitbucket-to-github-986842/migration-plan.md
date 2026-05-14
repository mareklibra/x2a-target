# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- Multiple interconnected services
- Security configurations that require careful migration
- External dependencies that need Ansible equivalents

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server configured to host multiple SSL-enabled virtual hosts with security hardening
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall)

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Vagrantfile`: Defines a Fedora 42 VM with port forwarding for development/testing. Migration considerations: Convert to Vagrant with Ansible provisioner or create equivalent Ansible-based development environment.
- `vagrant-provision.sh`: Shell script for Chef provisioning in Vagrant. Will be replaced by Ansible provisioning.
- `solo.rb` and `solo.json`: Chef Solo configuration files. Will be replaced by Ansible inventory and variable files.
- `Berksfile` and `Policyfile.rb`: Chef dependency management files. Dependencies will be managed through Ansible Galaxy requirements.

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible community.nginx role or create custom Nginx role
- **memcached (~> 6.0)**: Replace with community.general.memcached module or geerlingguy.memcached role
- **redisio (~> 7.2.4)**: Replace with community.general.redis module or geerlingguy.redis role
- **ssl_certificate (~> 2.1)**: Replace with Ansible crypto modules (openssl_certificate, openssl_privatekey)

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW. Migration should use ansible.posix.ufw module to maintain equivalent rules.
- **Fail2Ban Setup**: The Chef cookbook configures fail2ban. Migration should use community.general.fail2ban module.
- **SSH Hardening**: The Chef cookbook disables root login and password authentication. Migration should use ansible.posix.ssh module to maintain these settings.
- **SSL Certificate Management**: Self-signed certificates are generated for development. Migration should use Ansible's openssl modules.
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook (`fastapi`/`fastapi_password`)
  - These should be migrated to Ansible Vault or an external secrets management solution

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates multiple virtual hosts. The Ansible role will need to handle this dynamic configuration using templates and variables.
- **Service Orchestration**: The current setup has interdependent services (Nginx, Redis, Memcached, PostgreSQL, FastAPI). Ansible playbooks will need to maintain proper ordering and dependencies.
- **Security Hardening**: The Chef cookbooks implement various security measures that must be carefully migrated to maintain the security posture.
- **SSL Certificate Generation**: Self-signed certificates are generated for each site. This logic needs to be replicated in Ansible.

### Migration Order

1. **fastapi-tutorial** (Priority 1): This module is relatively self-contained and provides the application functionality.
2. **cache** (Priority 2): This module provides caching services needed by the application.
3. **nginx-multisite** (Priority 3): This module depends on the application being available and has the most complex configuration.

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions.
2. Self-signed certificates are acceptable for development; production would likely use different certificate sources.
3. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
5. The current Redis and PostgreSQL password configurations are for development only and would be replaced with secure passwords in production.
6. The current setup is designed for a single-node deployment; scaling considerations are not addressed.