# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies, configuration templates, and security settings. Based on the complexity and scope, this migration is estimated to require 3-4 weeks of effort with a team of 2 engineers.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site-specific configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, UFW firewall), sysctl security settings

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database provisioning, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with paths and log settings
- `Vagrantfile`: Defines a Fedora 42 VM with port forwarding and resource allocation
- `vagrant-provision.sh`: Shell script to install Chef and run the provisioning process

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for local development/testing

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role or community.general.memcached module
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or community.general.redis module

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW with specific rules for SSH, HTTP, and HTTPS
  - Migration approach: Use Ansible's `ufw` module to configure identical rules
  
- **Fail2ban Setup**: The Chef cookbook installs and configures fail2ban
  - Migration approach: Use Ansible's `package` module to install fail2ban and template module for configuration

- **SSH Hardening**: The Chef cookbook disables root login and password authentication
  - Migration approach: Use Ansible's `lineinfile` or template module to configure SSH settings

- **SSL Certificate Management**: Self-signed certificates are generated for each site
  - Migration approach: Use Ansible's `openssl_certificate` module to generate certificates

- **Vault/secrets management**: 
  - Redis password is hardcoded in the recipe (`redis_secure_password_123`)
  - PostgreSQL password is hardcoded in the recipe (`fastapi_password`)
  - Migration approach: Use Ansible Vault to store these credentials securely

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates site configurations based on node attributes
  - Mitigation: Create Ansible templates with Jinja2 loops to generate similar configurations

- **Redis Configuration Hack**: The Chef cookbook includes a ruby_block to modify Redis configuration files after they're created
  - Mitigation: Create a custom Redis configuration template in Ansible rather than modifying files after creation

- **Service Orchestration**: The Chef cookbook manages service dependencies (e.g., FastAPI depends on PostgreSQL)
  - Mitigation: Use Ansible handlers and meta dependencies to ensure proper service ordering

- **SSL Certificate Generation**: The Chef cookbook generates self-signed certificates for each site
  - Mitigation: Create an Ansible role for certificate management with proper idempotence checks

### Migration Order

1. **nginx-multisite cookbook** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Add security hardening features
   - Add multi-site configuration

2. **cache cookbook** (low complexity, standalone service)
   - Implement Memcached configuration
   - Implement Redis configuration with authentication

3. **fastapi-tutorial cookbook** (high complexity, application deployment)
   - Implement PostgreSQL database setup
   - Implement Python environment and application deployment
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora-based systems (the Vagrantfile specifies Fedora 42)
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or other CA)
3. The security requirements will remain the same (fail2ban, UFW, SSH hardening)
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. The Redis and Memcached configurations don't require advanced tuning beyond what's in the current recipes
6. The PostgreSQL database schema is managed by the FastAPI application, not by the infrastructure code
7. The Vagrant setup is primarily for development/testing and may not be needed in the final Ansible solution
8. No monitoring or logging solutions are currently implemented and won't be needed in the migration