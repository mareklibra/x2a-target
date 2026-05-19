# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

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
- Hardcoded credentials need to be migrated to Ansible Vault

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY


- **something-added**:
    - Description: foobar
    - Path: cookbooks/broken
    - Technology: Chef
    - Key Features: nothing

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints. Will be replaced by Ansible Galaxy requirements.yml.
- `Vagrantfile`: Defines the development VM using Fedora 42. Contains network configuration, resource allocation, and provisioning settings. Can be adapted for Ansible-based provisioning.
- `solo.json`: Contains the Chef run list and node attributes including Nginx site configurations and security settings. Will be converted to Ansible variables.
- `solo.rb`: Chef Solo configuration file. Defines cookbook paths and logging settings. Not needed in Ansible.
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef. Installs Chef and Berkshelf, then runs Chef Solo. Will be replaced with Ansible provisioning.

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or local development environments

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role `geerlingguy.nginx` or create a custom Nginx role
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or `DavidWittman.redis`

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW. Migrate to Ansible's `ufw` module or `firewalld` module depending on target OS.
- **fail2ban Setup**: Migrate fail2ban configuration to Ansible using the `template` module for configuration files.
- **SSH Hardening**: The cookbook disables root login and password authentication. Use Ansible's `lineinfile` module or a dedicated SSH role.
- **SSL Certificate Management**: Self-signed certificates are generated for development. Use Ansible's `openssl_certificate` module or consider integrating with Let's Encrypt via `geerlingguy.certbot`.
- **Vault/secrets management**:
  - Redis authentication password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL database credentials in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - Environment variables in .env file for FastAPI application
  - Total credentials detected: 3 hardcoded passwords/secrets

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations based on node attributes. Ansible will need to use templates with loops to achieve the same functionality.
- **SSL Certificate Generation**: Self-signed certificates are generated for each site. Ansible will need to handle this with the `openssl_certificate` module.
- **PostgreSQL User and Database Creation**: The Chef cookbook uses inline shell commands. Ansible has dedicated PostgreSQL modules that should be used instead.
- **Redis Configuration Hacks**: The Chef cookbook includes a ruby_block to modify Redis configuration files. Ansible should use templates with proper configuration instead of post-processing.
- **Service Management**: Multiple services (Nginx, Redis, Memcached, PostgreSQL, FastAPI) need to be managed. Ansible handlers should be used for service restarts.

### Migration Order

1. **something-added** (Priority 1): Supporting services


### Assumptions

1. The target environment will continue to be Fedora 42 or similar Linux distributions.
2. Self-signed certificates are acceptable for development, but production environments may require integration with Let's Encrypt or other certificate authorities.
3. The hardcoded passwords in the Chef cookbooks are for development only and will be replaced with Ansible Vault secrets in production.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git is accessible and will remain available.
5. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
6. The Nginx sites configuration (test.cluster.local, ci.cluster.local, status.cluster.local) will remain the same in the migrated environment.
7. The current VM resources (2GB RAM, 2 CPUs) are sufficient for the application stack.
8. No custom Chef resources or libraries are used that would require special handling in Ansible.
