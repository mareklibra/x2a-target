# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Memcached and Redis) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- **Total**: 5-7 weeks

**Complexity Assessment**: Medium
- Multiple interconnected services
- Security configurations that need careful migration
- External dependencies on Chef Supermarket cookbooks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
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
    - Key Features: Git-based deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with file paths and logging settings
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Vagrant configuration file for development environment using Fedora 42

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ mentioned in cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be targeting on-premises or local development environments

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible's `geerlingguy.memcached` role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible's `geerlingguy.redis` role or direct package installation and configuration

### Security Considerations

- **Firewall Configuration**: The current setup uses UFW; migration should adapt to target OS firewall (firewalld for Fedora/RHEL)
  - Migration approach: Use Ansible's `firewalld` module for RHEL/Fedora or `ufw` module for Ubuntu
  
- **Fail2ban Configuration**: Current setup includes fail2ban for brute force protection
  - Migration approach: Use Ansible to install and configure fail2ban with equivalent jail settings

- **SSH Hardening**: Current setup disables root login and password authentication
  - Migration approach: Use Ansible's `lineinfile` module or dedicated SSH role to apply equivalent security settings

- **Vault/secrets management**:
  - Redis password in plaintext in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL credentials in plaintext in the fastapi-tutorial cookbook (`fastapi:fastapi_password`)
  - Migration approach: Use Ansible Vault to encrypt sensitive values and reference them in playbooks

- **SSL Certificates**: Self-signed certificates are generated for each virtual host
  - Migration approach: Use Ansible's `openssl_*` modules to generate equivalent certificates or integrate with Let's Encrypt

### Technical Challenges

- **Multi-site Nginx Configuration**: The current setup dynamically creates Nginx virtual hosts from node attributes
  - Mitigation: Use Ansible templates with variable loops to create similar dynamic configurations

- **Redis Configuration Hacks**: The current setup includes a Ruby block to modify Redis configuration files after they're created
  - Mitigation: Create proper Redis configuration templates in Ansible rather than modifying files after creation

- **Service Dependencies**: The FastAPI application depends on PostgreSQL being configured first
  - Mitigation: Use Ansible's built-in dependency management with proper task ordering and handlers

- **SSL Certificate Management**: The current setup generates self-signed certificates
  - Mitigation: Use Ansible's `openssl_*` modules or integrate with Let's Encrypt for proper certificate management

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Supporting services that the application may depend on
   - Moderate complexity with external dependencies

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on other infrastructure components
   - Involves database setup and application configuration

### Assumptions

1. The target environment will continue to use Fedora or similar Linux distributions.
2. Self-signed certificates are acceptable for the migrated environment (production would likely need proper certificates).
3. The same security policies (SSH hardening, firewall rules) should be maintained in the Ansible version.
4. The FastAPI application source code will continue to be pulled from the same Git repository.
5. The directory structure for web content and application code will remain the same.
6. The current Redis and PostgreSQL passwords are development passwords and will be replaced with proper secrets management.
7. The Vagrant development environment will be maintained but provisioned with Ansible instead of Chef.