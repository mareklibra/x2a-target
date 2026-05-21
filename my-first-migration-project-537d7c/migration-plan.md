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
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite-something-else
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall)

- **renamed-fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database provisioning, systemd service configuration

- **something-added**:
    - Description: foo
    - Path: cookbooks/nginx-multisite-something-else-2
    - Technology: Chef
    - Key Features: nothing special

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints.
- `solo.json`: Chef Solo configuration file defining the run list and node attributes.
- `solo.rb`: Chef Solo configuration file specifying file paths and log settings.
- `Vagrantfile`: Defines a Vagrant VM configuration using Fedora 42 with port forwarding and resource allocation.
- `vagrant-provision.sh`: Shell script that installs Chef and Berkshelf, then runs Chef Solo in the Vagrant environment.

### Target Details

- **Operating System**: Fedora (based on Vagrantfile specifying "generic/fedora42"), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be targeting local development/testing environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role from Galaxy or direct package installation
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migration should maintain this capability while allowing for future integration with Let's Encrypt.
  - Migration approach: Use Ansible's `openssl_*` modules for certificate generation

- **Firewall Configuration**: The current implementation uses UFW.
  - Migration approach: Use Ansible's `ufw` module or `firewalld` module depending on target OS

- **SSH Hardening**: The current implementation disables root login and password authentication.
  - Migration approach: Use Ansible's `lineinfile` module or dedicated ssh hardening role

- **Fail2ban Configuration**: The current implementation installs and configures fail2ban.
  - Migration approach: Use Ansible's package module and templates for fail2ban configuration

- **Vault/secrets management**:
  - Redis password in plaintext in the cache cookbook (redis_secure_password_123)
  - PostgreSQL password in plaintext in the fastapi-tutorial cookbook (fastapi_password)
  - Migration approach: Use Ansible Vault for storing sensitive information

### Technical Challenges

- **Multi-site Nginx Configuration**: The current implementation dynamically generates site configurations based on node attributes.
  - Mitigation: Create Ansible templates with similar logic using with_items/loop constructs

- **Service Orchestration**: The current implementation manages service dependencies and notifications.
  - Mitigation: Use Ansible handlers and meta dependencies to maintain proper service restart ordering

- **PostgreSQL User/Database Management**: The current implementation uses shell commands to create users and databases.
  - Mitigation: Use Ansible's postgresql_* modules for more idiomatic database management

- **Python Environment Management**: The current implementation creates and manages Python virtual environments.
  - Mitigation: Use Ansible's pip module with virtualenv parameter

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first


### Assumptions

1. The target environment will continue to be Fedora-based, but with compatibility for Ubuntu and CentOS as specified in the cookbook metadata.
2. Self-signed certificates are acceptable for the migrated solution (production would likely require proper certificates).
3. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available and compatible.
5. The current plaintext passwords in the cookbooks are development/testing passwords and will be replaced with proper secret management in production.
6. The Vagrant development environment is primarily for testing and not part of the production deployment.
7. No custom Ohai plugins or Chef handlers are in use that would require special handling.
8. No Chef data bags or environments are in use beyond what's visible in the repository.
