# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks to Ansible roles and playbooks, addressing external dependencies, and ensuring security configurations are properly maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- Well-structured Chef cookbooks with clear dependencies
- Standard infrastructure components (Nginx, Redis, Memcached, PostgreSQL)
- Security configurations that need careful migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled subdomains, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening with fail2ban and UFW

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Chef policy configuration - will be replaced by Ansible playbook structure
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `solo.json`: Node attributes and run list - will be replaced by Ansible inventory and variables
- `Vagrantfile`: Development environment configuration - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible provisioning

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (from Vagrantfile) with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Libvirt (from Vagrantfile)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic cloud VMs

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management tasks

### Security Considerations

- **SSL Certificate Management**: Self-signed certificates are generated in the Chef cookbook. Ansible should maintain this capability while providing an option for proper CA-signed certificates.
- **Firewall Configuration**: UFW is configured in the Chef cookbook. Ansible should maintain this with appropriate firewall modules.
- **fail2ban Configuration**: Configured for brute force protection. Ansible should maintain this configuration.
- **SSH Hardening**: Root login and password authentication are disabled. Ansible should maintain these security practices.
- **System Hardening**: sysctl security configurations are applied. Ansible should maintain these settings.
- **Redis Authentication**: Redis is configured with a password. Ansible should maintain this security practice and implement proper secret management.
- **PostgreSQL Authentication**: Database credentials are stored in plaintext. Ansible should implement proper secret management.

### Technical Challenges

- **Multi-site Nginx Configuration**: The Chef cookbook dynamically creates Nginx site configurations based on attributes. Ansible will need to replicate this dynamic configuration approach.
- **SSL Certificate Generation**: Self-signed certificates are generated for each site. Ansible will need to handle certificate generation and management.
- **Service Dependencies**: The FastAPI application depends on PostgreSQL. Ansible will need to manage these dependencies properly.
- **Secret Management**: Several passwords are hardcoded in the Chef recipes. Ansible should use Ansible Vault or another secret management solution.

### Migration Order

1. **cache cookbook** (Low complexity, foundational service)
   - Implement Redis and Memcached configuration
   - Address authentication requirements

2. **nginx-multisite cookbook** (Medium complexity, core infrastructure)
   - Implement multi-site configuration
   - Implement SSL certificate management
   - Implement security hardening

3. **fastapi-tutorial cookbook** (High complexity, application layer)
   - Implement PostgreSQL configuration
   - Implement Python application deployment
   - Implement service management

### Assumptions

1. The target environment will continue to use the same operating systems (Fedora/Ubuntu/CentOS).
2. Self-signed certificates are acceptable for development, but production may require proper CA-signed certificates.
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
4. The current security configurations are appropriate and should be maintained.
5. The current service dependencies (Redis, Memcached, PostgreSQL) will remain the same.
6. The current network configuration (ports, protocols) will remain the same.
7. The current directory structure for web content (/var/www/[site]) will be maintained.
8. The current PostgreSQL database name and credentials can be maintained.
9. The current Redis password can be maintained but should be stored securely.