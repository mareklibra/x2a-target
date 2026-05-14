# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies on community cookbooks need to be replaced with Ansible Galaxy roles
- Security configurations and SSL certificate management require careful handling
- Secrets management needs to be implemented with Ansible Vault

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall), sysctl security settings

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints. Will be replaced by Ansible Galaxy requirements.yml.
- `solo.json`: Chef configuration file containing the run list and node attributes. Will be replaced by Ansible inventory variables.
- `solo.rb`: Chef Solo configuration file. Will be replaced by Ansible configuration.
- `Vagrantfile`: Defines the development VM using Fedora 42. Can be adapted for Ansible testing with minimal changes.
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks in the Vagrant environment. Will be replaced with Ansible provisioner.

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role `geerlingguy.nginx` or create a custom Nginx role
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or DavidWittman.redis

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW. Migration should use the Ansible `ufw` module or `firewalld` module depending on target OS.
- **Fail2ban Setup**: Currently configured in the nginx-multisite cookbook. Use Ansible to install and configure fail2ban with similar jail settings.
- **SSH Hardening**: The cookbook disables root login and password authentication. Implement using Ansible's `lineinfile` module or a dedicated SSH hardening role.
- **SSL Certificate Management**: Self-signed certificates are generated for development. Consider using Ansible's `openssl_*` modules or integrating with Let's Encrypt via `geerlingguy.certbot`.
- **Vault/secrets management**:
  - Redis password in cache cookbook: "redis_secure_password_123" (hardcoded)
  - PostgreSQL database credentials in fastapi-tutorial cookbook: username "fastapi" with password "fastapi_password" (hardcoded)
  - Environment variables in .env file for FastAPI application (hardcoded)
  - Total credentials detected: 3 sets of hardcoded credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations based on node attributes. Ansible templates will need to replicate this dynamic behavior.
- **SSL Certificate Generation**: Self-signed certificates are generated with specific attributes. Ensure Ansible's `openssl_*` modules can replicate this functionality.
- **Redis Configuration Hack**: The Chef cookbook includes a ruby_block to modify Redis configuration files after they're created. This will need a custom approach in Ansible.
- **Service Dependencies**: The FastAPI application depends on PostgreSQL. Ensure proper service ordering in Ansible playbooks.

### Migration Order

1. **nginx-multisite** (Priority 1): Core infrastructure component that other services depend on
   - Create base Nginx role
   - Implement security hardening (fail2ban, ufw, sysctl)
   - Configure SSL certificate generation
   - Implement multi-site configuration

2. **cache** (Priority 2): Supporting services
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Address the Redis configuration hack

3. **fastapi-tutorial** (Priority 3): Application layer
   - Implement PostgreSQL database setup
   - Configure Python environment and application deployment
   - Set up systemd service

### Assumptions

1. The target environment will continue to be Fedora 42 or a compatible Linux distribution.
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or provided certificates).
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
4. The hardcoded credentials in the Chef cookbooks are for development only and will be replaced with Ansible Vault secured variables.
5. The Nginx sites configuration (test.cluster.local, ci.cluster.local, status.cluster.local) will remain the same.
6. The current security settings (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
7. The Redis and Memcached configurations do not require significant changes beyond what's in the current cookbooks.