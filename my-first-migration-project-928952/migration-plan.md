# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application with PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- **Total**: 5-7 weeks

**Complexity Assessment**: Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies on community cookbooks need to be replaced with Ansible Galaxy roles
- Security configurations need careful migration
- Secrets management needs improvement in the Ansible implementation

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and self-signed certificate generation
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL/TLS setup, security hardening (fail2ban, ufw, sysctl)

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

- `Berksfile`: Chef dependency management file listing cookbook dependencies (both local and from Chef Supermarket)
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with paths and log settings
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Vagrant configuration for development environment using Fedora 42

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for local development/testing

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` module and configuration templates
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role for Memcached (e.g., `geerlingguy.memcached`)
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role for Redis (e.g., `geerlingguy.redis`)

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW; migrate to Ansible's `ufw` module
- **Fail2ban Setup**: Migrate fail2ban configuration to Ansible using the `template` module
- **SSH Hardening**: Migrate SSH security settings using Ansible's `lineinfile` or `template` modules
- **Sysctl Security Settings**: Migrate sysctl security configurations using Ansible's `sysctl` module
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook (`fastapi:fastapi_password`)
  - SSL certificates are generated on the fly with self-signed certificates
  - Recommendation: Use Ansible Vault for all credentials in the migrated solution

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of Nginx site configurations based on node attributes will need careful translation to Ansible's template system
- **SSL Certificate Generation**: The self-signed certificate generation logic needs to be migrated to Ansible's `openssl_certificate` module
- **Service Dependencies**: Ensuring proper ordering of service installations and configurations, especially for the FastAPI application which depends on PostgreSQL
- **Redis Configuration Hack**: The Chef cookbook includes a hack to fix Redis configuration files; this will need a clean implementation in Ansible

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Depends on external cookbooks that need to be replaced with Ansible Galaxy roles
   - Moderate complexity with Redis authentication configuration

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on the infrastructure being in place
   - Involves multiple components (Python, PostgreSQL, Git, systemd)

### Assumptions

1. The target environment will continue to be Fedora 42 as specified in the Vagrantfile
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or other CA)
3. The same security hardening approach will be maintained in the Ansible implementation
4. The FastAPI application source will continue to be pulled from the same Git repository
5. The current Redis and Memcached configurations are sufficient for the application needs
6. The current directory structure in the target system (/opt/fastapi-tutorial, /var/www/sites, etc.) will be maintained
7. No additional monitoring or logging solutions need to be implemented beyond what's in the current Chef configuration
8. The Vagrant development environment will be maintained for testing the Ansible playbooks