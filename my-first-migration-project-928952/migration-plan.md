# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies on community cookbooks need to be replaced with Ansible Galaxy roles
- Security configurations and SSL certificate management require careful migration
- Hardcoded secrets need to be addressed

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
    - Key Features: Git repository deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints. Will be replaced by Ansible Galaxy requirements.yml.
- `Vagrantfile`: Defines the development VM using Fedora 42. Contains network configuration, resource allocation, and provisioning instructions. Can be adapted for Ansible testing.
- `solo.json`: Chef run list and node attributes configuration. Contains site configurations and security settings. Will be converted to Ansible variables.
- `solo.rb`: Chef Solo configuration file. Defines cookbook paths and logging settings. Not needed in Ansible.
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks in the Vagrant environment. Will be replaced by Ansible provisioning.

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` module or community.general collection
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role for Memcached (e.g., geerlingguy.memcached)
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role for Redis (e.g., geerlingguy.redis)

### Security Considerations

- **SSL Certificate Management**: 
  - Current implementation generates self-signed certificates
  - Migration approach: Use Ansible's `openssl_*` modules for certificate generation
  - Consider integrating with Let's Encrypt for production environments

- **Firewall Configuration**:
  - Current implementation uses UFW
  - Migration approach: Use Ansible's `ufw` module or `firewalld` module depending on target OS

- **SSH Hardening**:
  - Current implementation disables root login and password authentication
  - Migration approach: Use Ansible's `lineinfile` module or dedicated SSH hardening role

- **Fail2ban Configuration**:
  - Current implementation installs and configures fail2ban
  - Migration approach: Use Ansible Galaxy role for fail2ban (e.g., geerlingguy.security)

- **Vault/secrets management**:
  - Credentials detected:
    - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
    - PostgreSQL user/password in fastapi-tutorial cookbook (hardcoded as 'fastapi'/'fastapi_password')
    - Database connection string in .env file
  - Migration approach: Use Ansible Vault for all credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Challenge: Dynamically generating multiple virtual host configurations with SSL
  - Mitigation: Use Ansible templates with loops over site configurations stored in variables

- **Service Dependencies**:
  - Challenge: Ensuring proper service startup order (PostgreSQL before FastAPI application)
  - Mitigation: Use Ansible handlers and meta dependencies between roles

- **SSL Certificate Management**:
  - Challenge: Proper handling of SSL certificates and private keys
  - Mitigation: Use Ansible Vault for sensitive files, implement proper file permissions

- **Redis Configuration Hacks**:
  - Challenge: The current implementation includes a hack to fix Redis configuration
  - Mitigation: Create proper Redis configuration template in Ansible

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Moderate complexity with security configurations and SSL management

2. **cache** (Priority 2)
   - Dependent services that the application will use
   - Lower complexity but requires proper security configuration for Redis

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on both web server and cache services
   - Higher complexity with database setup, application deployment, and service configuration

### Assumptions

1. The target environment will continue to be Fedora-based, though the cookbooks support Ubuntu and CentOS as well
2. Self-signed certificates are acceptable for development, but production would require proper certificates
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
4. The current security settings (disabling root login, password authentication, etc.) should be maintained
5. The current Redis and PostgreSQL passwords are for development only and will be replaced with secure passwords in production
6. The Nginx sites configuration in solo.json overrides the default attributes in the cookbook
7. The application is intended to run on port 8000 and be proxied through Nginx
8. The current implementation doesn't include backup or monitoring solutions