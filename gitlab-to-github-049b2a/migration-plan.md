# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated timeline for this migration is 3-4 weeks, with moderate complexity due to the security configurations and multiple service integrations.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and firewall configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, fail2ban integration, UFW firewall rules

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis authentication, Memcached configuration, service management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Chef policy definition specifying run list and cookbook versions - will be replaced by Ansible playbook structure
- `Vagrantfile`: VM configuration for development/testing - can be adapted for Ansible testing
- `solo.json`: Chef node attributes configuration - will be converted to Ansible variables
- `solo.rb`: Chef client configuration - not needed in Ansible
- `vagrant-provision.sh`: Shell script for provisioning Chef in Vagrant - will be replaced by Ansible provisioner in Vagrantfile

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package management
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package management
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management modules (openssl_*)

### Security Considerations

- **Firewall Configuration**: UFW rules need to be migrated to Ansible ufw module
  - SSH, HTTP, and HTTPS ports are allowed
  - Default deny policy is configured
  
- **Fail2ban Integration**: Configuration needs to be migrated to Ansible fail2ban module
  - Custom jail configuration is present

- **SSH Hardening**: SSH configuration needs to be migrated
  - Root login disabled
  - Password authentication disabled

- **System Hardening**: sysctl security settings need to be migrated

- **Vault/secrets management**:
  - Redis authentication password is hardcoded in the cache cookbook (redis_secure_password_123)
  - PostgreSQL database credentials are hardcoded in the fastapi-tutorial cookbook (fastapi/fastapi_password)
  - SSL certificates are generated on the fly with self-signed certificates
  - Total credentials detected: 2 hardcoded passwords

### Technical Challenges

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Ansible will need to replicate this or integrate with Let's Encrypt for production-ready certificates.
  - Mitigation: Use Ansible's openssl_* modules or community.crypto collection

- **Multi-site Configuration**: The nginx-multisite cookbook dynamically creates virtual hosts based on node attributes.
  - Mitigation: Use Ansible loops with templates to achieve similar functionality

- **Service Dependencies**: The FastAPI application depends on PostgreSQL, and the nginx configuration depends on the application being available.
  - Mitigation: Use Ansible handlers and proper task ordering to ensure dependencies are met

- **Firewall and Security**: The current implementation includes comprehensive security measures that must be carefully migrated.
  - Mitigation: Use Ansible's security-related modules and follow best practices for idempotent security configurations

### Migration Order

1. **cache** (Priority 1 - low complexity)
   - Simple service installation and configuration
   - Few dependencies
   - Clear configuration parameters

2. **fastapi-tutorial** (Priority 2 - moderate complexity)
   - Python application deployment
   - PostgreSQL database setup
   - Systemd service configuration

3. **nginx-multisite** (Priority 3 - higher complexity)
   - Multiple virtual hosts
   - SSL certificate generation
   - Security configurations
   - Depends on other services being available

### Assumptions

1. The target environment will continue to be Fedora/CentOS/Ubuntu based on the current configuration.
2. Self-signed certificates are acceptable for development, but production may require proper certificates.
3. The current security measures (fail2ban, ufw, SSH hardening) are required in the Ansible implementation.
4. The Redis and PostgreSQL passwords in the code are development passwords and will be replaced with Ansible Vault secured variables.
5. The FastAPI application source will continue to be pulled from the same Git repository.
6. The Vagrant development environment should be preserved with equivalent functionality.
7. The current directory structure for web content (/var/www/[site]) and SSL certificates (/etc/ssl/*) will be maintained.
8. The current implementation does not include backup or monitoring solutions, so these are not required in the migration.