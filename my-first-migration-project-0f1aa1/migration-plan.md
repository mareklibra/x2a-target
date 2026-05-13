# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Scope**: 3 Chef cookbooks with external dependencies
**Complexity**: Medium (standard web stack with common components)
**Estimated Timeline**: 2-3 weeks for complete migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and self-signed certificate generation
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL/TLS setup, security hardening (fail2ban, ufw firewall)

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database provisioning, systemd service configuration

### Infrastructure Files

- `Berksfile`: Chef dependency manager file listing cookbook dependencies (nginx, memcached, redisio)
- `solo.json`: Chef Solo configuration with run list and node attributes
- `solo.rb`: Chef Solo configuration file with cookbook paths
- `Vagrantfile`: Defines development VM using Fedora 42 with port forwarding and networking
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks in Vagrant

### Target Details

- **Operating System**: Fedora (based on Vagrantfile specifying "generic/fedora42")
- **Virtual Machine Technology**: Libvirt (specified in Vagrantfile)
- **Cloud Platform**: Not specified (appears to be local development environment)

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role `geerlingguy.nginx` or native Ansible modules
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or DavidWittman.redis

### Security Considerations

- **Firewall Configuration**: Migration of UFW rules to equivalent Ansible firewall modules
  - Approach: Use `ansible.posix.firewalld` module for Fedora
  
- **Fail2ban Setup**: Convert fail2ban configuration to Ansible
  - Approach: Use `geerlingguy.security` role or create custom tasks

- **SSH Hardening**: Migrate SSH security configurations
  - Approach: Use `dev-sec.ssh-hardening` role or `ansible.posix.sshd` module

- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL credentials in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - Database connection string in .env file
  - Count: 3 hardcoded credentials identified

### Technical Challenges

- **SSL Certificate Generation**: Chef cookbook generates self-signed certificates for each site
  - Mitigation: Use Ansible `openssl_*` modules to generate certificates or integrate with `geerlingguy.certbot` for Let's Encrypt

- **Multi-site Nginx Configuration**: Complex templating for multiple virtual hosts
  - Mitigation: Create Ansible templates based on existing Chef templates, use with_items/loop for site iteration

- **Redis Configuration Patching**: The Chef cookbook uses a ruby_block to modify Redis config files after installation
  - Mitigation: Use Ansible's lineinfile or template modules with proper configuration from the start

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Create base Nginx configuration first, then add virtual hosts and SSL

2. **cache** (Priority 2)
   - Independent service but required by the application
   - Relatively simple configuration with standard components

3. **fastapi-tutorial** (Priority 3)
   - Application layer that depends on both Nginx and database
   - More complex with database setup, Python environment, and application deployment

### Assumptions

1. The target environment will continue to be Fedora-based systems
2. Self-signed certificates are acceptable for the migrated solution (production would likely require proper certificates)
3. The same directory structure for web content will be maintained
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. No changes to the application configuration or database schema are required
6. The current security settings (fail2ban, ufw, SSH hardening) are appropriate for the target environment
7. Redis and Memcached configurations don't require performance tuning beyond defaults
8. No high availability or clustering is required for any services
9. No monitoring or logging solutions need to be integrated
10. The migration will not include CI/CD pipeline integration