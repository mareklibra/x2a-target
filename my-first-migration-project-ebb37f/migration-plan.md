# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application with PostgreSQL. The migration to Ansible will involve converting 3 Chef cookbooks with their recipes, templates, and attributes to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The codebase is well-structured with clear separation of concerns
- No complex custom resources or providers
- Standard infrastructure components (web server, caching, application deployment)
- Security configurations need careful attention during migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and SSL certificate management
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL/TLS setup with self-signed certificates, security hardening (fail2ban, UFW firewall), sysctl security settings

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database setup
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx ~> 12.0, memcached ~> 6.0, redisio ~> 7.2.4)
- `solo.json`: Defines the Chef run list and configuration attributes for nginx sites and security settings
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Defines the development VM (Fedora 42) with port forwarding and networking
- `vagrant-provision.sh`: Bash script to provision the VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for local development/testing

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks with custom configuration

### Security Considerations

- **SSL/TLS Configuration**: Migrate the self-signed certificate generation logic and secure TLS configuration
  - Migration approach: Use Ansible's openssl_* modules for certificate generation
  
- **Firewall (UFW)**: Migrate UFW firewall rules
  - Migration approach: Use Ansible's community.general.ufw module
  
- **fail2ban**: Migrate fail2ban configuration
  - Migration approach: Use Ansible templates for fail2ban configuration files
  
- **SSH Hardening**: Migrate SSH security settings (disable root login, password authentication)
  - Migration approach: Use Ansible's template module or lineinfile for sshd_config modifications
  
- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL user password in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - Migration approach: Use Ansible Vault for storing sensitive credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of multiple virtual hosts with SSL needs careful translation to Ansible
  - Mitigation: Use Ansible's with_items/loop constructs with templates to generate site configurations
  
- **Redis Configuration Hack**: The Chef cookbook includes a ruby_block to modify Redis configuration files after they're created
  - Mitigation: Create a custom Redis configuration template in Ansible that doesn't require post-processing
  
- **Service Dependencies**: Ensuring proper service ordering and dependencies in Ansible
  - Mitigation: Use handlers and notify mechanisms in Ansible to manage service restarts and reloads

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first
   
2. **cache** (Priority 2)
   - Supporting service with moderate complexity
   - Depends on proper network configuration from nginx-multisite
   
3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on both web server and database
   - Most complex due to application deployment, database setup, and service configuration

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as specified in the Vagrantfile)
2. The self-signed SSL certificates approach is acceptable for the migrated solution (not using Let's Encrypt or other CA)
3. The hardcoded passwords in the Chef recipes will be replaced with Ansible Vault variables
4. The same directory structure for web content and configuration files will be maintained
5. The Vagrant development environment will be preserved but updated to use Ansible provisioner instead of Chef
6. No changes to the application code or database schema are required during migration
7. The current security settings (fail2ban, UFW, SSH hardening) are appropriate and should be maintained
8. The FastAPI application will continue to be deployed from the same Git repository