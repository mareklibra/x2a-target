# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Scope**: 3 Chef cookbooks with external dependencies
**Complexity**: Medium (standard web stack with some security hardening)
**Estimated Timeline**: 2-3 weeks for complete migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and self-signed certificates
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
    - Key Features: Python virtual environment setup, PostgreSQL database provisioning, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with file paths and logging settings
- `Vagrantfile`: Defines a Fedora 42 VM for local development with port forwarding and networking
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks in the Vagrant environment

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified (appears to be designed for local development with Vagrant)

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role from Galaxy or custom role
- **memcached (~> 6.0)**: Replace with Ansible memcached role from Galaxy or custom role
- **redisio (~> 7.2.4)**: Replace with Ansible redis role from Galaxy or custom role

### Security Considerations

- **SSL Certificate Management**: 
  - Self-signed certificates are generated for each site
  - Migration approach: Use Ansible's openssl_* modules to generate certificates

- **Firewall Configuration**: 
  - UFW firewall is configured with specific rules
  - Migration approach: Use Ansible's ufw module for Ubuntu or firewalld module for Fedora/RHEL

- **SSH Hardening**:
  - Root login disabled
  - Password authentication disabled
  - Migration approach: Use Ansible's lineinfile or template module to configure SSH

- **Fail2ban Configuration**:
  - Fail2ban is installed and configured
  - Migration approach: Use Ansible's template module to configure fail2ban

- **Vault/secrets management**:
  - Redis password is hardcoded in the recipe
  - PostgreSQL password is hardcoded in the recipe
  - Migration approach: Use Ansible Vault to encrypt sensitive values

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The Chef cookbook dynamically creates multiple virtual hosts based on node attributes
  - Mitigation: Use Ansible's with_items/loop to iterate through site configurations

- **Redis Configuration Hack**: 
  - Description: The Chef cookbook includes a ruby_block to modify Redis configuration file
  - Mitigation: Use Ansible's lineinfile module or template with proper configuration

- **PostgreSQL User/Database Creation**: 
  - Description: The Chef cookbook uses shell commands to create database and user
  - Mitigation: Use Ansible's postgresql_* modules for proper idempotent database management

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for web services)
   - Create base Nginx role
   - Add SSL certificate generation
   - Add security hardening (fail2ban, ufw)
   - Add multi-site configuration

2. **cache** (low complexity, standalone services)
   - Create Memcached role
   - Create Redis role with authentication

3. **fastapi-tutorial** (high complexity, application deployment)
   - Create PostgreSQL role
   - Create Python application deployment role
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora 42 as specified in the Vagrantfile
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt)
3. The same security hardening measures should be applied in the Ansible solution
4. The FastAPI application source will continue to be pulled from the same Git repository
5. The Redis and PostgreSQL passwords should be secured using Ansible Vault in the migrated solution
6. The directory structure for web content (/var/www/[site]) should be maintained
7. The systemd service configuration for FastAPI should remain similar
8. The Vagrant development environment should be preserved with equivalent functionality