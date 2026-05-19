# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated timeline for this migration is 3-4 weeks, with moderate complexity due to the security configurations, SSL certificate management, and application deployment requirements.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **something-added**:
    - Description: foobar
    - Path: cookbooks/broken
    - Technology: Chef
    - Key Features: nothing special

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with file paths and log settings
- `Vagrantfile`: Vagrant configuration for local development using Fedora 42
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role
- **redisio (~> 7.2.4)**: Replace with Ansible redis role

### Security Considerations

- **SSL Certificate Management**: 
  - Self-signed certificates are generated for each site
  - Migration approach: Use Ansible's openssl_* modules or community.crypto collection

- **Firewall Configuration**: 
  - UFW is configured to allow SSH, HTTP, and HTTPS
  - Migration approach: Use Ansible's ufw module or firewalld module depending on target OS

- **SSH Hardening**: 
  - Root login disabled
  - Password authentication disabled
  - Migration approach: Use Ansible's lineinfile module or ssh_config module

- **System Hardening**: 
  - Sysctl security parameters
  - Migration approach: Use Ansible's sysctl module

- **Fail2ban Configuration**: 
  - Configured for SSH and web services
  - Migration approach: Use Ansible's template module or dedicated fail2ban role

- **Vault/secrets management**:
  - Redis password hardcoded in attributes (redis_secure_password_123)
  - PostgreSQL password hardcoded in recipe (fastapi_password)
  - Count: 2 hardcoded credentials identified

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current setup dynamically creates multiple virtual hosts with SSL
  - Mitigation: Use Ansible loops with templates to generate site configurations

- **SSL Certificate Generation**: 
  - Description: Self-signed certificates are generated for each site
  - Mitigation: Use Ansible's openssl_* modules to generate certificates or integrate with Let's Encrypt

- **Database Initialization**: 
  - Description: PostgreSQL database and user creation with proper permissions
  - Mitigation: Use Ansible's postgresql_* modules or community.postgresql collection

- **Application Deployment**: 
  - Description: Git-based deployment with Python virtual environment setup
  - Mitigation: Use Ansible's git module and pip module for Python dependencies

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation
   - Add security configurations (fail2ban, ufw, sysctl)
   - Implement SSL certificate generation
   - Configure virtual hosts

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, depends on database)
   - Set up PostgreSQL database
   - Deploy FastAPI application
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora-based, though the cookbooks support Ubuntu and CentOS as well
2. Self-signed certificates are acceptable for the migrated solution (vs. Let's Encrypt or other CA)
3. The security requirements (fail2ban, ufw, SSH hardening) will remain the same
4. The Redis and PostgreSQL passwords will be managed more securely in the Ansible solution
5. The FastAPI application source will continue to be pulled from the same Git repository
6. The Vagrant development environment will be maintained but converted to use Ansible provisioner
7. The current directory structure in the target environment (/opt/server/*, /etc/ssl/*) will be preserved
8. No additional monitoring or logging requirements beyond what's in the current Chef setup
