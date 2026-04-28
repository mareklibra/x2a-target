# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated complexity is medium, with an estimated timeline of 3-4 weeks for a complete migration.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and proper SSL configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, UFW firewall), system security settings

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external cookbook dependencies
- `solo.json`: Chef configuration file containing the run list and node attributes
- `solo.rb`: Chef configuration file specifying cookbook paths and log settings
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Vagrant configuration file for the development environment using Fedora 42

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be a local development environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role from Galaxy or custom role
- **memcached (~> 6.0)**: Replace with Ansible memcached role from Galaxy or custom role
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role from Galaxy or custom role

### Security Considerations

- **SSL Certificate Management**: 
  - Current implementation generates self-signed certificates
  - Migration approach: Use Ansible's openssl_* modules for certificate generation

- **Firewall Configuration**: 
  - Current implementation uses UFW
  - Migration approach: Use Ansible's ufw module or firewalld module (for Fedora)

- **Fail2ban Configuration**: 
  - Current implementation installs and configures fail2ban
  - Migration approach: Use Ansible's template module to configure fail2ban

- **System Security Settings**: 
  - Current implementation uses sysctl configurations
  - Migration approach: Use Ansible's sysctl module

- **Vault/secrets management**:
  - Redis password hardcoded in recipe
  - PostgreSQL credentials hardcoded in recipe and .env file
  - Migration approach: Use Ansible Vault for storing sensitive credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation dynamically creates multiple virtual hosts based on node attributes
  - Mitigation strategy: Use Ansible loops with templates to achieve similar functionality

- **SSL Certificate Generation**: 
  - Description: Self-signed certificates are generated for each site
  - Mitigation strategy: Use Ansible's openssl_* modules with proper idempotency checks

- **PostgreSQL User and Database Creation**: 
  - Description: Current implementation uses shell commands for database operations
  - Mitigation strategy: Use Ansible's postgresql_* modules for better idempotency and error handling

- **Python Environment Management**: 
  - Description: Current implementation creates and configures Python virtual environments
  - Mitigation strategy: Use Ansible's pip module with virtualenv parameter

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Create base Nginx role
   - Implement virtual host configuration
   - Implement SSL certificate generation
   - Implement security configurations

2. **cache** (low complexity, standalone service)
   - Create Memcached role
   - Create Redis role with authentication

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - Create PostgreSQL role
   - Create Python application deployment role
   - Implement systemd service configuration

### Assumptions

1. The target environment will continue to be Fedora-based systems
2. Self-signed certificates are acceptable for the migrated solution
3. The same security measures (fail2ban, UFW, SSH hardening) are required in the new implementation
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. The current Redis and PostgreSQL passwords are development passwords and will be replaced with proper secrets management
6. The current directory structure in the target environment (/opt/fastapi-tutorial, /var/www/sites) will be maintained
7. The Vagrant development environment will be maintained for testing the Ansible playbooks