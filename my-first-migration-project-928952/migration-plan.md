# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Memcached and Redis) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies on community cookbooks need to be replaced with Ansible equivalents
- Security configurations and SSL certificate management require careful handling
- Secrets management needs to be implemented with Ansible Vault

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw, sysctl)

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

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints. Will be replaced by Ansible Galaxy requirements.yml.
- `Vagrantfile`: Defines the development VM using Fedora 42. Will need to be updated to use Ansible provisioner instead of Chef.
- `solo.json`: Contains the Chef run list and configuration data. Will be converted to Ansible group_vars or host_vars.
- `solo.rb`: Chef configuration file. No direct Ansible equivalent needed.
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks. Will be replaced with Ansible provisioning.

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or local development environments

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role from Galaxy or create a custom role
- **memcached (~> 6.0)**: Replace with Ansible memcached role from Galaxy or create a custom role
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role from Galaxy or create a custom role

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migration should:
  - Maintain the same certificate generation capability for development
  - Add support for Let's Encrypt or other certificate authorities for production
  - Ensure proper permissions on private keys

- **Firewall Configuration**: The current implementation uses UFW. Migration should:
  - Use the Ansible UFW module for Ubuntu targets
  - Use the Ansible firewalld module for Fedora/CentOS targets
  - Maintain the same port allowances (SSH, HTTP, HTTPS)

- **Fail2ban Configuration**: Migrate the fail2ban configuration to use Ansible's fail2ban module

- **SSH Hardening**: Maintain the SSH security configurations:
  - Disable root login
  - Disable password authentication

- **System Hardening**: Migrate the sysctl security configurations

- **Vault/secrets management**: 
  - Redis password in cache cookbook: "redis_secure_password_123"
  - PostgreSQL user/password in fastapi-tutorial cookbook: "fastapi"/"fastapi_password"
  - Database connection string in fastapi-tutorial .env file
  - All credentials should be moved to Ansible Vault

### Technical Challenges

- **Multi-platform Support**: The current cookbooks support both Ubuntu and CentOS. The Ansible roles will need to handle differences between distributions (package names, service names, file paths).
  - Mitigation: Use Ansible's facts and conditionals to handle distribution-specific tasks

- **SSL Certificate Generation**: The current implementation uses inline shell commands to generate SSL certificates.
  - Mitigation: Use Ansible's openssl_* modules for certificate management

- **PostgreSQL Configuration**: The current implementation uses shell commands to create database users and permissions.
  - Mitigation: Use Ansible's postgresql_* modules for database management

- **Redis Configuration Hack**: The current implementation includes a ruby_block to modify Redis configuration files after they're created.
  - Mitigation: Use Ansible templates to generate correct Redis configuration files directly

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Create Ansible roles for:
     - nginx base configuration
     - virtual host management
     - SSL certificate management
     - security hardening (fail2ban, ufw, sysctl)

2. **cache** (Priority 2)
   - Dependent services that the application will need
   - Create Ansible roles for:
     - Memcached configuration
     - Redis installation and configuration with authentication

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on the infrastructure
   - Create Ansible roles for:
     - Python environment setup
     - PostgreSQL database configuration
     - Application deployment from Git
     - Systemd service configuration

### Assumptions

1. The target environment will continue to be Fedora 42 or similar Linux distributions.
2. Self-signed certificates are acceptable for development environments.
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
4. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
5. The Redis and PostgreSQL passwords in the current configuration are development passwords and will be replaced with secure passwords in production.
6. The Nginx sites configuration in solo.json overrides the default attributes in the cookbook.
7. The application will continue to run on port 8000 and be proxied through Nginx.
8. The current VM resource allocation (2GB RAM, 2 CPUs) is sufficient for the application.