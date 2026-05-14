# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies, configuration templates, and security settings.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3-4 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7-8 weeks

**Complexity Assessment:** Medium
- Well-structured Chef cookbooks with clear separation of concerns
- Standard infrastructure components (Nginx, Redis, Memcached, PostgreSQL)
- Security configurations that need careful migration
- Hardcoded credentials that should be moved to Ansible Vault

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and SSL certificate management
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL/TLS setup, security headers, fail2ban integration, UFW firewall configuration

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with paths and log settings
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Vagrant configuration file for development environment using Fedora 42

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in Vagrant development environment
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role or service module
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or service module

### Security Considerations

- **SSL/TLS Configuration**: 
  - Migration approach: Use Ansible crypto modules for certificate generation
  - Ensure proper handling of private keys with restricted permissions

- **Firewall (UFW)**:
  - Migration approach: Use Ansible ufw module to configure firewall rules
  - Maintain the same security posture with default deny and specific allow rules

- **fail2ban**:
  - Migration approach: Use Ansible to install and configure fail2ban
  - Maintain the same jail configurations

- **SSH Hardening**:
  - Migration approach: Use Ansible ssh_config module to apply the same security settings
  - Maintain root login restrictions and password authentication settings

- **Vault/secrets management**:
  - Redis password: Currently hardcoded in the cache cookbook
  - PostgreSQL credentials: Currently hardcoded in the fastapi-tutorial cookbook
  - Migration approach: Move all credentials to Ansible Vault
  - Count of credentials detected: 2 (Redis password, PostgreSQL user/password)

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation uses Chef templates to generate multiple virtual host configurations
  - Mitigation strategy: Create Ansible templates with similar structure and use with_items to iterate through site configurations

- **SSL Certificate Generation**:
  - Description: Self-signed certificates are generated using OpenSSL commands
  - Mitigation strategy: Use Ansible openssl_* modules to generate certificates with proper permissions

- **Service Orchestration**:
  - Description: The current implementation manages service dependencies and notifications
  - Mitigation strategy: Use Ansible handlers and meta dependencies to ensure proper service restart ordering

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Supporting services that the application will use
   - Moderate complexity with Redis authentication

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on both nginx and cache services
   - Higher complexity with database setup and application configuration

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS/RHEL-based distributions
2. Self-signed certificates are acceptable for the migrated environment (production would likely use proper certificates)
3. The same security posture is desired in the Ansible implementation
4. The Vagrant development environment should be preserved with similar functionality
5. No changes to the application code or database schema are required
6. The current hardcoded credentials will be replaced with Ansible Vault variables
7. The same directory structure for web content and application code will be maintained
8. The same service user accounts will be used in the migrated environment