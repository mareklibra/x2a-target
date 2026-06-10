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
- Multiple interconnected services
- Security configurations that need careful migration
- External dependencies on Chef Supermarket cookbooks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening with fail2ban and UFW

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git-based deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with file paths and log settings
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Vagrant configuration file for local development environment using Fedora 42

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ mentioned in cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or collection (e.g., `ansible.posix.nginx`)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., `geerlingguy.memcached`)
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (e.g., `geerlingguy.redis`)

### Security Considerations

- **Firewall Configuration**: UFW rules need to be migrated to appropriate firewall modules
  - Migration approach: Use `ansible.posix.firewalld` for Fedora/CentOS and `community.general.ufw` for Ubuntu
  
- **Fail2ban Configuration**: Fail2ban setup needs to be migrated
  - Migration approach: Use `community.general.fail2ban` module

- **SSH Hardening**: SSH configuration hardening needs to be preserved
  - Migration approach: Use `ansible.posix.sshd_config` module

- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL password in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - SSL certificates and private keys in nginx-multisite cookbook
  - Migration approach: Use Ansible Vault for storing sensitive values

### Technical Challenges

- **Multi-platform Support**: The current Chef cookbooks support both Ubuntu and CentOS/RHEL-based systems
  - Mitigation: Use Ansible's conditionals and variables based on `ansible_os_family` to handle platform-specific tasks

- **SSL Certificate Generation**: The current implementation generates self-signed certificates
  - Mitigation: Use Ansible's `community.crypto.openssl_*` modules to generate certificates or integrate with Let's Encrypt

- **Dynamic Site Configuration**: The nginx-multisite cookbook dynamically creates site configurations based on node attributes
  - Mitigation: Use Ansible templates with loops to generate site configurations based on variables

- **Redis Configuration Hacks**: The cache cookbook includes a Ruby block to modify Redis configuration files
  - Mitigation: Create proper Redis configuration templates in Ansible

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Supporting services that the application will need
   - Moderate complexity with external dependencies

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on the infrastructure being in place
   - Contains database setup and application-specific configurations

### Assumptions

1. The target environment will continue to use Fedora 42 or a compatible Linux distribution
2. Self-signed SSL certificates are acceptable for the migrated solution (production would likely use proper certificates)
3. The same security policies (fail2ban, UFW, SSH hardening) should be maintained
4. The FastAPI application source code will remain available at the same Git repository
5. The current hardcoded passwords will be replaced with Ansible Vault secured variables
6. The Vagrant development environment should be preserved but updated to use Ansible provisioning
7. No changes to the application architecture or deployment model are required
8. The current Chef-based solution is functional and the Ansible migration should maintain feature parity