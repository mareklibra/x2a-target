# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their recipes, templates, and attributes to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has well-structured Chef cookbooks with clear dependencies
- Security configurations are present and need careful migration
- Multiple services need to be coordinated (Nginx, Redis, Memcached, PostgreSQL, FastAPI)

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configurations
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
    - Key Features: Git repository deployment, Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx, memcached, redisio) - will be replaced by Ansible Galaxy requirements.yml
- `Vagrantfile`: Defines development VM using Fedora 42 - can be adapted for Ansible testing
- `solo.json`: Defines Chef run list and configuration attributes - will be replaced by Ansible inventory variables
- `solo.rb`: Chef configuration file - not needed in Ansible
- `vagrant-provision.sh`: Shell script to install Chef and run cookbooks - will be replaced by Ansible provisioner in Vagrantfile

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role from Galaxy or direct package installation
- **memcached (~> 6.0)**: Replace with Ansible memcached role from Galaxy or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible redis role from Galaxy or direct package installation

### Security Considerations

- **SSL Certificate Management**: 
  - Self-signed certificates are generated for each site
  - Migration should maintain the same certificate paths and permissions
  - Consider using Ansible's openssl_* modules for certificate generation

- **Firewall Configuration**: 
  - UFW firewall is configured with specific rules
  - Replace with Ansible's ufw module or firewalld for Fedora

- **Fail2ban Integration**:
  - Custom jail configuration is applied
  - Use Ansible's template module to create equivalent configuration

- **SSH Hardening**:
  - Root login disabled
  - Password authentication disabled
  - Use Ansible's lineinfile or template module to configure sshd_config

- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook (redis_secure_password_123)
  - PostgreSQL password is hardcoded in the fastapi-tutorial cookbook (fastapi_password)
  - These should be moved to Ansible Vault or an external secrets manager

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - The Chef cookbook dynamically creates multiple virtual hosts
  - Ansible implementation will need to use loops with the template module
  - Ensure proper SSL certificate generation for each site

- **Service Coordination**: 
  - Dependencies between services (PostgreSQL before FastAPI, etc.)
  - Use Ansible handlers and proper task ordering to ensure services start in correct order

- **Redis Configuration**: 
  - Current implementation includes a hack to fix Redis configuration
  - Ansible implementation should provide a clean template without requiring post-processing

- **Python Environment Management**:
  - Virtual environment creation and dependency installation
  - Use Ansible's pip module with virtualenv parameter

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Start with basic Nginx installation and configuration
   - Add SSL and security features
   - Finally implement multi-site configuration

2. **cache** (Priority 2)
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Ensure proper service management

3. **fastapi-tutorial** (Priority 3)
   - Set up PostgreSQL database
   - Deploy FastAPI application from Git
   - Configure Python environment and dependencies
   - Create systemd service

### Assumptions

1. The target environment will continue to be Fedora 42 or similar Linux distributions
2. Self-signed certificates are acceptable (no integration with Let's Encrypt or external CA)
3. The same directory structure for web content will be maintained
4. The FastAPI application repository will remain available at the specified URL
5. No changes to the application configuration or database schema are required
6. The current security settings (firewall rules, fail2ban, SSH hardening) are appropriate for the target environment
7. Redis and Memcached configurations don't require advanced clustering or replication
8. No monitoring or logging solutions are currently implemented that need migration