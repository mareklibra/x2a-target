# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The codebase is well-structured with clear separation of concerns
- Security configurations are comprehensive and must be carefully migrated
- External dependencies on community cookbooks need Ansible equivalents

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and custom site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw)

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, PostgreSQL database provisioning, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Chef policy definition - will be replaced by Ansible playbooks
- `solo.json`: Chef node configuration - will be migrated to Ansible inventory variables
- `solo.rb`: Chef configuration - will be replaced by ansible.cfg
- `Vagrantfile`: Development environment definition - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisioning script - will be replaced by Ansible playbooks

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role (e.g., geerlingguy.nginx)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., geerlingguy.memcached)
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (e.g., geerlingguy.redis)
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management modules (openssl_*)

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migrate to Ansible's openssl_* modules for certificate generation and management.
- **Firewall Configuration (UFW)**: Migrate UFW rules to Ansible's ufw module, maintaining the same security posture.
- **Fail2ban Configuration**: Ensure fail2ban configurations are properly migrated using Ansible's template module.
- **SSH Hardening**: Maintain SSH security configurations (disable root login, password authentication) using Ansible's lineinfile or template modules.
- **Redis Authentication**: Ensure Redis password is securely managed, potentially using Ansible Vault for the password.
- **PostgreSQL Security**: Migrate database user creation and permission management to Ansible's postgresql_* modules.

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of multiple virtual hosts with SSL needs careful implementation in Ansible using templates and loops.
- **Service Orchestration**: Ensuring proper service restart notifications when configurations change.
- **SSL Certificate Generation**: Implementing self-signed certificate generation in Ansible while maintaining proper file permissions.
- **Secrets Management**: Moving hardcoded passwords (Redis, PostgreSQL) to Ansible Vault.
- **Python Environment Management**: Ensuring proper Python virtual environment setup and dependency installation.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for web services)
   - Start with basic Nginx installation and configuration
   - Implement SSL certificate generation
   - Configure virtual hosts
   - Implement security hardening (fail2ban, ufw)

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, depends on database)
   - Set up PostgreSQL database
   - Configure Python environment
   - Deploy FastAPI application
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as indicated in the Vagrantfile).
2. Self-signed certificates are acceptable for the migrated solution (production environments might require proper CA-signed certificates).
3. The security posture (firewall rules, SSH hardening, etc.) should be maintained at the same level.
4. The current Redis password and PostgreSQL credentials are development values and will be replaced with proper secrets management.
5. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
6. The current directory structure in the target environment (/opt/server/*, /var/www/*) should be maintained.
7. The Vagrant development environment will be maintained for testing the Ansible playbooks.