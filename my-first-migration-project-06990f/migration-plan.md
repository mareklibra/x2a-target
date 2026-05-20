# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site web server with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated complexity is moderate, with security configurations and multiple service integrations requiring careful attention.

**Timeline Estimate:**
- Planning and preparation: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and validation: 1-2 weeks
- Documentation and knowledge transfer: 1 week
- Total: 5-7 weeks

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and fail2ban integration
    - Path: cookbooks/nginx-multisite22
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security headers, firewall configuration

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis password protection, Memcached configuration

- **something-added**:
    - Description: nothing
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis password protection, Memcached configuration


### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies including nginx (~> 12.0), memcached (~> 6.0), and redisio (~> 7.2.4). Migration will require identifying equivalent Ansible Galaxy roles or creating custom roles.
- `solo.json`: Contains the run list and configuration attributes for the Chef deployment. Will need to be converted to Ansible inventory variables.
- `Vagrantfile`: Defines the development VM (Fedora 42) with port forwarding and networking. Can be adapted for Ansible testing with minimal changes.
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef. Will need to be replaced with Ansible provisioning.

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or create a custom role based on the official Ansible documentation
- **memcached (~> 6.0)**: Use `geerlingguy.memcached` Ansible role or create a custom role
- **redisio (~> 7.2.4)**: Use `geerlingguy.redis` or `DavidWittman.redis` Ansible role with appropriate configuration

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migration should maintain this capability while allowing for integration with Let's Encrypt or other certificate providers.
- **Firewall Configuration**: UFW rules need to be migrated to equivalent Ansible UFW module tasks.
- **fail2ban Integration**: Configuration needs to be migrated to Ansible tasks.
- **SSH Hardening**: Current configuration disables root login and password authentication. These settings need to be preserved in the Ansible implementation.
- **Security Headers**: Nginx security headers configuration needs to be maintained in templates.
- **Vault/secrets management**:
  - Redis password in cache cookbook: 1 hardcoded password in the recipe
  - PostgreSQL credentials in fastapi-tutorial cookbook: 2 hardcoded passwords in the recipe
  - Consider using Ansible Vault for these credentials

### Technical Challenges

- **Multi-site Configuration**: The dynamic generation of Nginx site configurations based on attributes will need to be replicated using Ansible's templating system.
- **Service Integration**: Coordinating the deployment of multiple interconnected services (Nginx, Redis, Memcached, PostgreSQL, FastAPI application) will require careful planning of role dependencies.
- **SSL Certificate Generation**: The self-signed certificate generation logic will need to be replicated in Ansible.
- **Python Environment Management**: The Python virtual environment setup and dependency installation will need to be handled by Ansible's Python modules.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Create base Nginx role
   - Implement SSL certificate generation
   - Configure security features (fail2ban, UFW)
   - Implement multi-site configuration

2. **cache** (low complexity, standalone service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - Implement PostgreSQL database setup
   - Configure Python environment
   - Deploy application from Git
   - Set up systemd service

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions.
2. Self-signed certificates are acceptable for development/testing, but production deployment may require integration with a certificate authority.
3. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
5. The current hardcoded credentials in the recipes are for development purposes and will be replaced with more secure methods in production.
6. The current Nginx configuration with multiple virtual hosts will be maintained in the Ansible implementation.
7. The current Redis and Memcached configurations are appropriate for the application's caching needs.
