# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:** 3-4 weeks
**Complexity:** Medium
**Team Size Recommendation:** 2-3 DevOps engineers

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall), sysctl security settings

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

- `Berksfile`: Chef dependency manager file listing cookbook dependencies (nginx, ssl_certificate, memcached, redisio)
- `Policyfile.rb`: Chef policy file defining the run list and cookbook dependencies
- `Vagrantfile`: Vagrant configuration for local development/testing using Fedora 42
- `solo.rb`: Chef Solo configuration file
- `solo.json`: Chef node attributes and run list configuration
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) based on cookbook metadata, with Fedora 42 used for Vagrant testing
- **Virtual Machine Technology**: Vagrant with libvirt provider (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role (e.g., geerlingguy.nginx or custom role)
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management tasks using openssl module
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., geerlingguy.memcached)
- **redisio (~> 7.2.4)**: Replace with Ansible redis role (e.g., geerlingguy.redis)

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migration should maintain this capability while allowing for integration with Let's Encrypt or other certificate providers.
- **Firewall Configuration**: UFW firewall rules need to be migrated to equivalent Ansible UFW module tasks.
- **fail2ban Configuration**: Current fail2ban setup needs to be migrated to Ansible tasks.
- **SSH Hardening**: SSH security settings (disable root login, password authentication) need to be preserved.
- **Vault/secrets management**:
  - Redis password in cache cookbook: "redis_secure_password_123" (hardcoded in recipe)
  - PostgreSQL database credentials in fastapi-tutorial cookbook: username "fastapi" with password "fastapi_password" (hardcoded in recipe)
  - Environment variables in .env file for FastAPI application (DATABASE_URL contains credentials)

### Technical Challenges

- **Multi-site Nginx Configuration**: The current implementation uses Chef templates and attributes to configure multiple Nginx sites. Ansible will need equivalent template handling.
- **SSL Certificate Generation**: Self-signed certificate generation logic needs to be preserved in Ansible.
- **Security Hardening**: Comprehensive security measures (fail2ban, ufw, sysctl) need to be properly migrated.
- **Service Orchestration**: The current implementation manages multiple services (Nginx, Redis, Memcached, PostgreSQL, FastAPI application). Proper service ordering and dependencies need to be maintained.
- **Python Application Deployment**: The FastAPI application deployment involves Git, virtual environments, and systemd service configuration that needs careful migration.

### Migration Order

1. **nginx-multisite** (Priority 1): Core infrastructure component that other services depend on
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Implement security hardening (fail2ban, ufw)
   - Configure virtual hosts

2. **cache** (Priority 2): Supporting services
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (Priority 3): Application layer
   - PostgreSQL database setup
   - Python environment and application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS as specified in the cookbook metadata.
2. Self-signed certificates are acceptable for the migrated solution (production environments might require integration with proper certificate authorities).
3. The security requirements (fail2ban, ufw, SSH hardening) will remain the same in the Ansible implementation.
4. The FastAPI application source code will continue to be available at the specified Git repository.
5. The current Redis and PostgreSQL passwords are development/testing passwords and should be replaced with proper secret management in production.
6. The Vagrant setup is primarily for testing and may not be required in the final Ansible implementation if other testing methods are preferred.