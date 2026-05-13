# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3-4 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7-8 weeks

**Complexity Assessment:** Medium
- Multiple interconnected services
- Security configurations that need careful migration
- External dependencies on community cookbooks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configurations
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
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Manages cookbook dependencies - will be replaced by Ansible Galaxy requirements file
- `solo.json`: Contains node configuration and attributes - will be replaced by Ansible inventory and group_vars
- `solo.rb`: Chef configuration file - will be replaced by Ansible configuration
- `Vagrantfile`: Defines development VM - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisions the Vagrant VM with Chef - will be replaced with Ansible provisioning

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or local development

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or collection (e.g., `ansible.posix.nginx`)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., `geerlingguy.memcached`)
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (e.g., `geerlingguy.redis`)

### Security Considerations

- **Firewall Configuration**: UFW rules need to be migrated to appropriate Ansible firewall modules
  - Migration approach: Use `ansible.posix.firewalld` or `community.general.ufw` modules
  
- **Fail2ban Setup**: Configuration needs to be preserved
  - Migration approach: Use `community.general.fail2ban` module or templates for configuration

- **SSH Hardening**: SSH configuration for root access and password authentication
  - Migration approach: Use `ansible.posix.sshd` module or templates for sshd_config

- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL credentials in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - SSL certificates and private keys in nginx-multisite cookbook
  - Migration approach: Use Ansible Vault for all credentials and sensitive data

### Technical Challenges

- **SSL Certificate Generation**: The Chef cookbook generates self-signed certificates
  - Migration strategy: Use `community.crypto.openssl_*` modules to generate certificates or integrate with Let's Encrypt

- **Multi-site Configuration**: The nginx-multisite cookbook dynamically creates site configurations
  - Migration strategy: Use Ansible templates with loops to generate site configurations from variables

- **Service Dependencies**: Ensuring proper ordering of service installation and configuration
  - Migration strategy: Use Ansible handlers and proper task dependencies

- **PostgreSQL Configuration**: Database and user creation for FastAPI application
  - Migration strategy: Use `community.postgresql` collection for database management

### Migration Order

1. **cache** (Priority 1 - low complexity)
   - Simple configuration of Memcached and Redis
   - Few dependencies

2. **nginx-multisite** (Priority 2 - medium complexity)
   - Core web server functionality
   - Security configurations
   - SSL certificate management

3. **fastapi-tutorial** (Priority 3 - higher complexity)
   - Application deployment
   - Database configuration
   - Depends on web server and potentially cache services

### Assumptions

1. The target environment will continue to use Fedora as the base OS
2. Self-signed certificates are acceptable for development/testing
3. The same directory structure for web content will be maintained
4. The FastAPI application repository will remain available at the same URL
5. The security requirements (fail2ban, firewall, SSH hardening) will remain the same
6. Redis and Memcached configurations don't require advanced tuning beyond what's in the current cookbooks
7. PostgreSQL will be installed locally on the same server as the FastAPI application