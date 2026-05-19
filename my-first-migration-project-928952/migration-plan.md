# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7 weeks

**Complexity Assessment:** Medium
- The codebase is well-structured with clear separation of concerns
- External dependencies are clearly defined
- Security configurations are present and need careful migration
- Multiple services with interdependencies require coordinated deployment

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security headers, fail2ban integration, UFW firewall configuration

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

- `Berksfile`: Defines cookbook dependencies including nginx (~> 12.0), memcached (~> 6.0), and redisio (~> 7.2.4). Migration will require mapping these to Ansible Galaxy roles or creating custom roles.
- `solo.json`: Contains the run list and configuration attributes for the Chef deployment. Will need to be converted to Ansible inventory variables.
- `solo.rb`: Chef configuration file that defines paths and log settings. Ansible equivalent would be ansible.cfg.
- `Vagrantfile`: Defines the development VM using Fedora 42. Can be reused with minor modifications to use Ansible provisioner instead of Chef.
- `vagrant-provision.sh`: Shell script that installs Chef and runs the cookbooks. Will need to be replaced with Ansible installation and playbook execution.

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for local development/testing

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role `geerlingguy.nginx` or create a custom Nginx role
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or DavidWittman.redis

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migration should maintain this functionality or improve it with Let's Encrypt integration.
  - Migration approach: Use Ansible's `openssl_*` modules for certificate generation or integrate with `geerlingguy.certbot` for Let's Encrypt.

- **Firewall Configuration**: UFW is configured with specific rules for HTTP, HTTPS, and SSH.
  - Migration approach: Use Ansible's `ufw` module to maintain identical firewall rules.

- **fail2ban Integration**: Configured to protect services from brute force attacks.
  - Migration approach: Create an Ansible role for fail2ban configuration using templates.

- **SSH Hardening**: Root login and password authentication are disabled.
  - Migration approach: Use Ansible's `lineinfile` module or the `ansible-hardening` role.

- **Vault/secrets management**: 
  - Redis password in cache cookbook: `redis_secure_password_123`
  - PostgreSQL credentials in fastapi-tutorial cookbook: username `fastapi` with password `fastapi_password`
  - Migration approach: Use Ansible Vault to encrypt sensitive values and reference them in playbooks.

### Technical Challenges

- **Multi-site Nginx Configuration**: The current implementation dynamically generates site configurations based on attributes.
  - Mitigation: Create Ansible templates with Jinja2 loops to generate similar configurations from variables.

- **Service Interdependencies**: The FastAPI application depends on PostgreSQL, and the web server depends on the application being available.
  - Mitigation: Use Ansible handlers and the `notify` mechanism to ensure services are restarted in the correct order.

- **SSL Certificate Generation**: Self-signed certificates are generated with specific attributes.
  - Mitigation: Use Ansible's `openssl_certificate` module with similar parameters.

- **Custom Redis Configuration**: The current implementation includes a hack to fix Redis configuration.
  - Mitigation: Create a custom Redis configuration template in Ansible that addresses these issues directly.

### Migration Order

1. **cache** (Priority 1): Relatively simple configuration for Memcached and Redis services.
2. **fastapi-tutorial** (Priority 2): Application deployment with database dependencies.
3. **nginx-multisite** (Priority 3): Most complex module with security configurations and SSL management.

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as specified in the Vagrantfile).
2. Self-signed certificates are acceptable for the migrated solution (production environments might require proper CA-signed certificates).
3. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate and should be maintained.
4. The FastAPI application source will continue to be pulled from the same Git repository.
5. The current passwords in the Chef recipes (`redis_secure_password_123` and `fastapi_password`) are development/testing passwords and should be replaced with proper secrets management in production.
6. The Vagrant development environment should be maintained for testing the Ansible playbooks.
7. No additional services beyond what's currently configured will be needed in the initial migration.