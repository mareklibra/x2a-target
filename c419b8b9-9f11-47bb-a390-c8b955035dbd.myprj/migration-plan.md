# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks to Ansible roles and playbooks, addressing external dependencies, and ensuring security configurations are properly maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The codebase is well-structured with clear separation of concerns
- Security configurations are comprehensive but straightforward
- External dependencies are clearly defined
- No complex custom resources or providers are used

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site-specific configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall)

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Chef policy definition - will be replaced by Ansible playbook structure
- `solo.json`: Chef node configuration - will be converted to Ansible inventory variables
- `solo.rb`: Chef solo configuration - no direct Ansible equivalent needed
- `Vagrantfile`: VM configuration for development - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisioning script - will be replaced by Ansible playbook

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role (e.g., geerlingguy.nginx)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., geerlingguy.memcached)
- **redisio (~> 7.2.4)**: Replace with Ansible redis role (e.g., geerlingguy.redis)
- **ssl_certificate (~> 2.1)**: Replace with Ansible's openssl_* modules for certificate management

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW. Migrate to Ansible's `ufw` module or `firewalld` module depending on target OS.
- **Fail2ban Setup**: Migrate fail2ban configuration to Ansible using the `template` module for configuration files.
- **SSL Certificate Management**: The cookbook generates self-signed certificates. Use Ansible's `openssl_certificate` module.
- **SSH Hardening**: The cookbook disables root login and password authentication. Use Ansible's `lineinfile` or `template` module to configure SSH.
- **Redis Authentication**: The cookbook sets a Redis password. Ensure this is securely managed in Ansible Vault.
- **PostgreSQL Authentication**: Database credentials are hardcoded. Store these in Ansible Vault.

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations. Implement using Ansible templates and loops.
- **Service Orchestration**: The Chef cookbook manages service dependencies. Ensure proper ordering in Ansible using handlers and dependencies.
- **SSL Certificate Generation**: The Chef cookbook generates self-signed certificates. Implement using Ansible's openssl modules.
- **Python Environment Management**: The Chef cookbook creates and manages Python virtual environments. Use Ansible's `pip` module with virtualenv parameter.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation
   - Add SSL certificate management
   - Implement multi-site configuration
   - Add security hardening

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, depends on database)
   - Implement PostgreSQL database setup
   - Implement Python application deployment
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora-based systems (the Vagrantfile specifies Fedora 42).
2. Self-signed certificates are acceptable for development/testing (production would likely use Let's Encrypt or other CA).
3. The security requirements (fail2ban, firewall, SSH hardening) will remain the same.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
5. The Redis password and PostgreSQL credentials will need to be securely stored in Ansible Vault.
6. The current directory structure in the target environment (/var/www/, /opt/fastapi-tutorial) will be maintained.
7. The Nginx site configurations (test.cluster.local, ci.cluster.local, status.cluster.local) will remain the same.
8. The current port mappings and networking configuration will be preserved.