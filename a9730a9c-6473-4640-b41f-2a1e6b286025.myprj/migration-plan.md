# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for deploying a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1 week
- Documentation and Knowledge Transfer: 1 week
- Total: 5-6 weeks

**Complexity:** Medium to High
- Multiple interconnected services
- Security-focused configuration
- SSL certificate management
- Database integration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall)

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

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements
- `Policyfile.rb` and `Policyfile.lock.json`: Chef policy definitions - will be replaced by Ansible playbooks
- `Vagrantfile`: VM configuration for development/testing - can be adapted for Ansible testing
- `solo.rb` and `solo.json`: Chef Solo configuration - will be replaced by Ansible inventory and variables
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible provisioning

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (from Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Libvirt (from Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **memcached (~> 6.0)**: Replace with Ansible memcached role (geerlingguy.memcached or similar)
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (geerlingguy.redis or similar)
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management tasks or community role

### Security Considerations

- **SSL Certificate Management**: Migration must maintain proper certificate generation and permissions
  - Self-signed certificates are currently generated with OpenSSL
  - Proper file permissions (640) and ownership (root:ssl-cert) must be preserved

- **Firewall Configuration**: UFW firewall rules must be migrated
  - Default deny policy
  - Allow SSH, HTTP, HTTPS

- **Fail2ban Integration**: Configuration must be preserved
  - Current setup uses a custom jail.local template

- **SSH Hardening**: Security settings must be maintained
  - Root login disabled
  - Password authentication disabled

- **System Hardening**: Sysctl security settings must be migrated
  - Custom sysctl-security.conf template

- **Redis Authentication**: Password protection must be maintained
  - Current password: "redis_secure_password_123" (should be changed and stored securely)

- **PostgreSQL Security**: Database credentials must be handled securely
  - Current credentials: User "fastapi" with password "fastapi_password" (should be changed and stored securely)

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of multiple virtual hosts with SSL needs careful implementation in Ansible
  - Solution: Use Ansible templates with loops over site configurations

- **SSL Certificate Management**: Self-signed certificate generation and proper permissions
  - Solution: Create an Ansible role for certificate management with proper file permissions

- **Service Orchestration**: Ensuring services start in the correct order with proper dependencies
  - Solution: Use Ansible handlers and meta dependencies between roles

- **Security Hardening**: Maintaining the comprehensive security measures
  - Solution: Create dedicated security role with firewall, fail2ban, and system hardening tasks

- **Database Integration**: Setting up PostgreSQL with proper user/database creation
  - Solution: Use Ansible PostgreSQL modules or community role with proper credential management

### Migration Order

1. **Base Infrastructure** (Low complexity)
   - System packages and basic configuration
   - Security hardening (firewall, fail2ban, SSH)

2. **Nginx Configuration** (Medium complexity)
   - Base Nginx installation
   - SSL certificate generation
   - Virtual host configuration

3. **Caching Services** (Medium complexity)
   - Memcached configuration
   - Redis installation and security

4. **FastAPI Application** (High complexity)
   - PostgreSQL database setup
   - Python environment configuration
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions
2. Self-signed certificates are acceptable (no integration with Let's Encrypt or other CA)
3. The same security policies should be maintained in the Ansible implementation
4. The FastAPI application source will continue to be pulled from the same Git repository
5. Redis and PostgreSQL passwords in the current implementation are for development only and will be replaced with secure values in production
6. The current multi-site configuration (test.cluster.local, ci.cluster.local, status.cluster.local) will be maintained
7. The Vagrant development environment will continue to be used for testing