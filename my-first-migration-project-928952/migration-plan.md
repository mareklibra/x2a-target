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
- The repository has well-structured Chef cookbooks with clear separation of concerns
- External dependencies on community cookbooks need to be replaced with Ansible Galaxy roles
- Security configurations and SSL certificate management require careful migration
- Credential management needs to be improved during migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
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
    - Key Features: Git repository deployment, Python virtual environment setup, PostgreSQL database provisioning, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints.
- `solo.json`: Chef Solo configuration file containing the run list and node attributes.
- `solo.rb`: Chef Solo configuration file specifying cookbook paths and log settings.
- `Vagrantfile`: Defines the development VM using Fedora 42 with port forwarding and resource allocation.
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef Solo.

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in the Vagrant development environment.
- **Virtual Machine Technology**: Vagrant with libvirt provider (based on Vagrantfile configuration).
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic cloud VMs.

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role from Galaxy or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible's `geerlingguy.memcached` role or custom Ansible tasks
- **redisio (~> 7.2.4)**: Replace with Ansible's `geerlingguy.redis` role or custom Ansible tasks

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW firewall. Migration should use Ansible's `firewalld` or `ufw` modules.
  - Migration approach: Use Ansible's `ufw` module to configure the same rules
  
- **Fail2ban Setup**: The Chef cookbook installs and configures fail2ban.
  - Migration approach: Use Ansible's package and template modules to install and configure fail2ban

- **SSH Hardening**: The Chef cookbook disables root login and password authentication.
  - Migration approach: Use Ansible's `lineinfile` module or the `ansible.posix.sshd` module

- **SSL Certificate Management**: Self-signed certificates are generated for each site.
  - Migration approach: Use Ansible's `openssl_certificate` module to generate self-signed certificates

- **Vault/secrets management**:
  - Redis password is hardcoded in the recipe (`redis_secure_password_123`)
  - PostgreSQL credentials are hardcoded in the FastAPI recipe (`fastapi:fastapi_password`)
  - Migration approach: Move all credentials to Ansible Vault and use variables

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations based on node attributes.
  - Mitigation: Create Ansible templates with Jinja2 loops to generate site configurations from variables

- **Redis Configuration Hack**: The Chef cookbook includes a Ruby block to modify Redis configuration files after they're created.
  - Mitigation: Create a custom Redis configuration template in Ansible that doesn't require post-processing

- **PostgreSQL User and Database Creation**: The Chef cookbook uses inline shell commands to create PostgreSQL users and databases.
  - Mitigation: Use Ansible's `postgresql_user` and `postgresql_db` modules for cleaner implementation

- **FastAPI Application Deployment**: The Chef cookbook clones a Git repository and sets up a Python environment.
  - Mitigation: Use Ansible's `git` module and Python-related modules for a more idiomatic implementation

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Supports the application but has external dependencies
   - Moderate complexity with Redis configuration requiring special attention

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on the infrastructure being in place
   - Involves database setup, application deployment, and service configuration

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS/RHEL-based systems.
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or other CA).
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
4. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
5. The Vagrant development environment will be maintained for testing the Ansible playbooks.
6. The current Redis and Memcached configurations meet performance requirements and don't need optimization.
7. No monitoring or logging solutions are currently implemented and won't be added during migration.
8. The current PostgreSQL setup doesn't include replication or backup configurations.