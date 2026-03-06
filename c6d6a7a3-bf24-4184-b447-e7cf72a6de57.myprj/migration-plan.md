# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI Python application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks, handling external dependencies, and ensuring security configurations are properly maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The codebase is well-structured with clear separation of concerns
- Security configurations are comprehensive and will need careful migration
- Multiple services with interdependencies will require coordinated testing

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server configured to host multiple SSL-enabled websites with robust security configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL/TLS setup with self-signed certificates, security hardening (fail2ban, UFW firewall), system-level security configurations

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation and configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies including nginx (~12.0), memcached (~6.0), and redisio (~7.2.4)
- `Policyfile.rb`: Defines the run list and cookbook dependencies for Chef Policyfile workflow
- `Vagrantfile`: Configures a Fedora 42 VM with networking and provisioning for development/testing
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `solo.rb`: Chef Solo configuration file defining cookbook paths and log settings
- `solo.json`: JSON configuration for Chef Solo with site configurations and security settings

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~12.0)**: Replace with Ansible nginx role (e.g., geerlingguy.nginx or custom role)
- **memcached (~6.0)**: Replace with Ansible memcached role (e.g., geerlingguy.memcached)
- **redisio (~7.2.4)**: Replace with Ansible Redis role (e.g., geerlingguy.redis or DavidWittman.redis)
- **ssl_certificate (~2.1)**: Replace with Ansible certificate management tasks using openssl module

### Security Considerations

- **Firewall (UFW)**: Migrate UFW rules to Ansible ufw module
- **fail2ban**: Configure fail2ban using Ansible fail2ban role or custom tasks
- **SSL/TLS Configuration**: Ensure proper certificate generation and secure TLS settings
- **System Hardening**: Migrate sysctl security settings using Ansible sysctl module
- **SSH Hardening**: Maintain SSH security configurations (disable root login, password authentication)

### Technical Challenges

- **Multi-site Nginx Configuration**: Ensure proper templating of Nginx site configurations with SSL support
- **Redis Configuration**: Address the custom Redis configuration "hack" in the cache cookbook
- **Service Dependencies**: Maintain proper ordering of service deployments (database before application)
- **Certificate Management**: Properly handle SSL certificate generation and permissions
- **Security Headers**: Ensure all security headers are properly configured in Nginx

### Migration Order

1. Base Infrastructure (low risk)
   - System packages
   - Firewall configuration
   - System security settings

2. Database Services (moderate complexity)
   - PostgreSQL installation and configuration
   - Database and user creation

3. Caching Services (moderate complexity)
   - Memcached configuration
   - Redis installation with authentication

4. Web Server (moderate complexity)
   - Nginx installation
   - SSL certificate generation
   - Site configurations

5. Application Deployment (high complexity)
   - Python environment setup
   - FastAPI application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42)
2. Self-signed certificates are acceptable for development/testing (production would likely use Let's Encrypt or other CA)
3. The security requirements will remain the same (fail2ban, UFW, SSH hardening)
4. The FastAPI application repository will remain available at the specified URL
5. Redis and Memcached configurations will maintain the same basic parameters
6. The multi-site configuration pattern will be preserved
7. No additional monitoring or logging solutions are required beyond what's in the current configuration
8. The migration will not introduce new features or architectural changes