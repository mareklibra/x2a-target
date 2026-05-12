# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Memcached and Redis) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The codebase is well-structured with clear separation of concerns
- Multiple external dependencies need to be addressed
- Security configurations require careful migration
- Secrets management needs improvement in the Ansible implementation

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configuration
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
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints. Will be replaced by Ansible Galaxy requirements.yml.
- `solo.json`: Chef run list and node attributes configuration. Will be replaced by Ansible inventory variables.
- `solo.rb`: Chef Solo configuration file. Will be replaced by Ansible configuration.
- `Vagrantfile`: Defines the development VM using Fedora 42. Can be adapted for Ansible testing with minimal changes.
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks. Will be replaced with Ansible provisioner in Vagrant.

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in the Vagrant development environment.
- **Virtual Machine Technology**: Vagrant with libvirt provider (based on Vagrantfile configuration).
- **Cloud Platform**: Not specified in the repository. The configuration appears to be designed for on-premises or generic VM deployment.

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible's `geerlingguy.memcached` role or custom implementation
- **redisio (~> 7.2.4)**: Replace with Ansible's `geerlingguy.redis` role or custom implementation
- **Python 3 and venv**: Use Ansible's `pip` and `package` modules for Python environment setup
- **PostgreSQL**: Use Ansible's `postgresql` modules or `geerlingguy.postgresql` role

### Security Considerations

- **SSL Certificate Management**: 
  - Current implementation generates self-signed certificates
  - Migration approach: Use Ansible's `openssl_*` modules or integrate with `geerlingguy.certbot` for Let's Encrypt

- **Firewall Configuration (UFW)**:
  - Migration approach: Use Ansible's `ufw` module to replicate the current rules

- **Fail2ban Configuration**:
  - Migration approach: Use Ansible's `template` module to create fail2ban configuration files

- **SSH Hardening**:
  - Migration approach: Use Ansible's `lineinfile` module or `ansible.posix.sshd` module to configure SSH security settings

- **Sysctl Security Settings**:
  - Migration approach: Use Ansible's `sysctl` module to apply kernel parameter security settings

- **Vault/secrets management**:
  - Identified credentials:
    - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
    - PostgreSQL user and password in fastapi-tutorial cookbook (hardcoded as 'fastapi' and 'fastapi_password')
    - Database connection string in .env file
  - Migration approach: Use Ansible Vault to encrypt sensitive values

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Challenge: Dynamically generating multiple virtual host configurations with SSL
  - Mitigation: Create a flexible Ansible role with templates and variable structures similar to the current Chef attributes

- **Redis Configuration Hacks**: 
  - Challenge: The current implementation includes a Ruby block to modify Redis configuration files after they're created
  - Mitigation: Create proper Redis configuration templates in Ansible rather than modifying files after creation

- **Service Orchestration**: 
  - Challenge: Ensuring proper service restart/reload only when needed
  - Mitigation: Use Ansible handlers and notify mechanism to replicate Chef's notification system

- **PostgreSQL User and Database Creation**:
  - Challenge: The current implementation uses direct shell commands
  - Mitigation: Use Ansible's PostgreSQL modules for idempotent database management

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Implement security hardening features
   - Configure multi-site virtual hosts

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - Set up PostgreSQL database
   - Deploy FastAPI application from Git
   - Configure Python environment and dependencies
   - Create systemd service

### Assumptions

1. The target environment will continue to be either Ubuntu (>= 18.04) or CentOS (>= 7.0)
2. Self-signed certificates are acceptable for development, but production may require proper certificates
3. The current security configurations are appropriate for the target environment
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. The hardcoded credentials in the current implementation are for development only and will be replaced with secure values in production
6. The Vagrant development environment will continue to be used for testing
7. No custom Chef resources or libraries are being used that would require special handling
8. The current implementation does not include monitoring or logging solutions that would need migration