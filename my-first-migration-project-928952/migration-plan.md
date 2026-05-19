# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3-4 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7-8 weeks

**Complexity Assessment:** Medium
- Multiple interconnected services
- Security configurations that need careful migration
- External dependencies on Chef Supermarket cookbooks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site configurations
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
    - Key Features: Python virtual environment setup, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Manages cookbook dependencies, including external dependencies from Chef Supermarket. Will need to be replaced with Ansible Galaxy requirements.
- `Vagrantfile`: Defines a Fedora 42 VM for development/testing with port forwarding and resource allocation. Can be adapted for Ansible testing.
- `solo.json`: Contains Chef node attributes and run list. Will need to be converted to Ansible group_vars or host_vars.
- `solo.rb`: Chef Solo configuration file. Will be replaced by Ansible configuration.
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef. Will need to be updated for Ansible provisioning.

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or local development environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or collection (e.g., `ansible.posix.nginx`)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., `geerlingguy.memcached`)
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (e.g., `geerlingguy.redis`)

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW. Migration should use Ansible's `ansible.posix.firewalld` or `community.general.ufw` modules.
  - Migration approach: Convert UFW rules to Ansible ufw module tasks

- **Fail2ban Configuration**: The Chef cookbook configures fail2ban. Migration should use Ansible's `community.general.fail2ban` module.
  - Migration approach: Convert fail2ban jail configuration to Ansible templates

- **SSH Hardening**: The Chef cookbook disables root login and password authentication. Migration should maintain these security practices.
  - Migration approach: Use Ansible's `ansible.posix.sshd_config` module

- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL password in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - SSL certificates and private keys in nginx-multisite cookbook
  - Total credentials detected: 3 (2 hardcoded passwords, 1 SSL certificate configuration)

### Technical Challenges

- **SSL Certificate Management**: The Chef cookbook generates self-signed certificates. Migration should maintain this functionality or integrate with Ansible's `community.crypto` collection.
  - Mitigation strategy: Use Ansible's `community.crypto.openssl_*` modules to generate certificates

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations. Migration should maintain this flexibility.
  - Mitigation strategy: Use Ansible templates with loops to generate site configurations

- **Redis Configuration Patching**: The Chef cookbook includes a hack to fix Redis configuration. Migration should address this properly.
  - Mitigation strategy: Use Ansible templates with proper configuration options instead of post-configuration patching

- **Service Orchestration**: The Chef cookbook manages service dependencies. Migration should maintain proper service ordering.
  - Mitigation strategy: Use Ansible handlers and meta dependencies to ensure proper service ordering

### Migration Order

1. **nginx-multisite** (Priority 1): Foundation for web services, other components depend on it
2. **cache** (Priority 2): Supports the application but has external dependencies
3. **fastapi-tutorial** (Priority 3): Application deployment that depends on other components

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as indicated in the Vagrantfile)
2. Self-signed SSL certificates are acceptable for the migrated solution
3. The current security configurations (fail2ban, UFW, SSH hardening) are required in the migrated solution
4. The FastAPI application source will continue to be pulled from the same Git repository
5. The current Redis and PostgreSQL passwords are development/testing passwords and should be replaced with Ansible Vault encrypted values in production
6. The Nginx site configurations in solo.json will be maintained in the Ansible inventory
7. The current VM specifications (2GB RAM, 2 CPUs) are sufficient for the application stack
8. Port forwarding requirements (80→8080, 443→8443) will remain the same