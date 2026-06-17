# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Scope**: 3 Chef cookbooks with external dependencies
**Complexity**: Medium
**Estimated Timeline**: 3-4 weeks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site configurations
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

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible requirements.yml
- `Vagrantfile`: VM configuration for development/testing - can be adapted for Ansible testing
- `solo.json`: Chef node attributes and run list - will be replaced by Ansible inventory variables
- `solo.rb`: Chef configuration - will be replaced by ansible.cfg
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible playbook

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ mentioned in cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or dedicated tasks for Redis installation and configuration

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW - migrate to Ansible's firewall modules (ufw or firewalld depending on target OS)
- **Fail2ban Setup**: Migrate fail2ban configuration to Ansible tasks
- **SSH Hardening**: Preserve SSH security settings (disable root login, password authentication)
- **SSL Certificate Management**: Migrate self-signed certificate generation to Ansible
- **Vault/secrets management**:
  - Redis password hardcoded in attributes (`redis_secure_password_123`)
  - PostgreSQL credentials hardcoded in recipe (`fastapi:fastapi_password`)
  - Consider migrating to Ansible Vault for secure credential storage

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of multiple virtual hosts will need careful translation to Ansible templates
- **SSL Certificate Management**: Self-signed certificate generation logic needs to be preserved
- **Service Dependencies**: Ensuring proper ordering of service installations and configurations (e.g., PostgreSQL before FastAPI application)
- **Idempotency**: Ensuring all custom commands remain idempotent in Ansible (particularly database user creation)

### Migration Order

1. **nginx-multisite cookbook** (moderate complexity, foundation for web services)
   - Start with basic Nginx installation and configuration
   - Add security hardening components
   - Implement SSL certificate generation
   - Configure virtual hosts

2. **cache cookbook** (low complexity, standalone services)
   - Implement Memcached configuration
   - Implement Redis installation and configuration

3. **fastapi-tutorial cookbook** (high complexity, application deployment)
   - Set up PostgreSQL database
   - Deploy FastAPI application
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora-based systems, with potential for Ubuntu/CentOS as mentioned in cookbook metadata
2. The Vagrant development environment will be maintained
3. Self-signed certificates are acceptable for development (production would likely use Let's Encrypt or other CA)
4. The current security configurations are appropriate for the target environment
5. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
6. The Redis configuration hack in the cache cookbook is addressing compatibility issues that may need investigation during migration
7. No custom Ohai plugins or Chef handlers are in use that would require special handling
8. No Chef data bags or encrypted data are being used beyond what's visible in the repository