# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup with three main cookbooks: nginx-multisite, cache, and fastapi-tutorial. The migration to Ansible will involve converting these cookbooks to Ansible roles and playbooks while maintaining the same functionality and security configurations.

**Scope**: 3 Chef cookbooks with dependencies on external cookbooks
**Complexity**: Medium (standard web stack with caching and application deployment)
**Estimated Timeline**: 2-3 weeks for complete migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and proper SSL configuration
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

- `Berksfile`: Dependency management file listing both local and external cookbook dependencies
  - Migration consideration: Replace with Ansible Galaxy requirements.yml
- `solo.json`: Node configuration with run list and attributes
  - Migration consideration: Convert to Ansible group_vars or host_vars
- `solo.rb`: Chef Solo configuration
  - Migration consideration: Replace with ansible.cfg
- `Vagrantfile`: Defines development VM configuration
  - Migration consideration: Update to use Ansible provisioner instead of Chef
- `vagrant-provision.sh`: Shell script to install Chef and run cookbooks
  - Migration consideration: Replace with Ansible playbook

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be targeting local development environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation

### Security Considerations

- **SSL Certificate Management**: 
  - Current approach: Self-signed certificates generated with OpenSSL
  - Migration approach: Use Ansible's openssl_* modules or community.crypto collection

- **Firewall Configuration**: 
  - Current approach: UFW configuration via Chef
  - Migration approach: Use Ansible's ufw module or firewalld module for Fedora

- **Fail2ban Integration**: 
  - Current approach: Custom fail2ban jail configuration
  - Migration approach: Use Ansible's template module with fail2ban configuration templates

- **Vault/secrets management**:
  - Redis password hardcoded in recipe: "redis_secure_password_123"
  - PostgreSQL password hardcoded in recipe: "fastapi_password"
  - Migration approach: Use Ansible Vault for storing sensitive credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Challenge: Maintaining the dynamic site configuration capability
  - Mitigation: Use Ansible's with_items/loop constructs with templates to generate site configurations

- **SSL Certificate Generation**: 
  - Challenge: Ensuring certificates are only generated when needed
  - Mitigation: Use Ansible's stat module to check for existing certificates before generation

- **Service Dependencies**: 
  - Challenge: Ensuring proper service startup order (PostgreSQL before FastAPI)
  - Mitigation: Use Ansible's handlers and meta dependencies to manage service ordering

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Moderate complexity with Redis and Memcached configuration
   - Dependent services but not as critical as web server

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on both web server and database
   - Most complex with multiple components (Python, PostgreSQL, application code)

### Assumptions

1. The target environment will continue to be Fedora-based systems
2. Self-signed certificates are acceptable for the migrated solution
3. The same security hardening measures should be maintained
4. The FastAPI application repository will remain available at the specified URL
5. The current directory structure in the target environment (/opt/server/*, /var/www/*) should be preserved
6. The current network configuration (ports, IP addresses) should be maintained
7. No additional monitoring or logging requirements beyond what's in the current Chef setup