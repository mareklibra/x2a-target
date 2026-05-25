# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3-4 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7-8 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies are explicitly defined
- Security configurations are present and need careful migration
- Multiple services need to be coordinated (Nginx, Redis, Memcached, PostgreSQL, FastAPI)

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall)

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

- `Berksfile`: Defines cookbook dependencies (both local and external from Chef Supermarket). Will be replaced by Ansible Galaxy requirements.yml.
- `Vagrantfile`: Defines the development VM environment using Fedora 42. Can be adapted for Ansible testing.
- `solo.json`: Contains Chef run list and node attributes. Will be converted to Ansible inventory variables.
- `solo.rb`: Chef Solo configuration. Will be replaced by Ansible configuration.
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef. Will be replaced by Ansible provisioning.

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) as specified in cookbook metadata, with Fedora 42 used in the Vagrant development environment.
- **Virtual Machine Technology**: Vagrant with libvirt provider as indicated in the Vagrantfile.
- **Cloud Platform**: Not specified in the repository. The configuration appears to be designed for on-premises or generic cloud VMs.

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible's `geerlingguy.memcached` role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible's `geerlingguy.redis` role or direct package installation and configuration

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW with specific rules for SSH, HTTP, and HTTPS. Ansible will need to use the `ufw` module to replicate this configuration.
- **fail2ban Setup**: The Chef cookbook installs and configures fail2ban. Ansible will need to use the `package` module and templates to configure fail2ban similarly.
- **SSH Hardening**: The Chef cookbook disables root login and password authentication. Ansible will need to use the `lineinfile` or `template` module to configure SSH similarly.
- **SSL Certificate Management**: The Chef cookbook generates self-signed certificates for development. Ansible will need to use the `openssl_certificate` module to generate certificates.
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL database credentials are hardcoded in the fastapi-tutorial cookbook (`fastapi:fastapi_password`)
  - No Chef Vault or encrypted data bags are used in the current implementation

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations based on node attributes. Ansible will need to use templates and loops to achieve similar functionality.
- **Service Coordination**: The FastAPI application depends on PostgreSQL being configured first. Ansible will need to ensure proper ordering of tasks.
- **SSL Certificate Generation**: The Chef cookbook generates self-signed certificates for each site. Ansible will need to replicate this functionality using the `openssl_certificate` module.
- **Redis Configuration Hack**: The Chef cookbook includes a Ruby block to modify Redis configuration files after they're created. Ansible will need to use templates or the `lineinfile` module to achieve similar results.

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Secondary infrastructure component
   - Relatively simple configuration with external dependencies

3. **fastapi-tutorial** (Priority 3)
   - Application-specific configuration
   - Depends on proper infrastructure setup

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS as specified in the cookbook metadata.
2. The self-signed certificates are for development only and not production use.
3. The hardcoded passwords in the cookbooks are not production values and will be replaced with Ansible Vault variables.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
5. The Vagrant development environment will be maintained for testing the Ansible playbooks.
6. The current security configurations (fail2ban, ufw, SSH hardening) are sufficient and will be replicated in Ansible.
7. The current service configurations (Nginx, Redis, Memcached, PostgreSQL) are sufficient and will be replicated in Ansible.