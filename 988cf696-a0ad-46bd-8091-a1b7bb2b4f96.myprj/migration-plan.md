# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Memcached and Redis) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3-4 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7-8 weeks

**Complexity Assessment:** Medium to High
- Multiple interconnected services
- Security configurations that need careful migration
- Database and application deployment with specific requirements

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server configured to host multiple SSL-enabled websites with security hardening
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
    - Key Features: Python virtual environment setup, Git-based deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (both local and external) - will be replaced by Ansible Galaxy requirements
- `Policyfile.rb` and `Policyfile.lock.json`: Chef policy definitions - will be replaced by Ansible playbooks
- `solo.json`: Configuration data for Chef Solo - will be replaced by Ansible variables
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `Vagrantfile`: Defines the development VM - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible provisioning

### Target Details

Based on the source repository analysis:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ mentioned in cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **ssl_certificate (~> 2.1)**: Replace with Ansible crypto modules for certificate management
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package configuration

### Security Considerations

- **SSL Certificate Management**: The current setup generates self-signed certificates. Migrate to Ansible crypto modules or consider integrating with Let's Encrypt for production.
- **Firewall Configuration (ufw)**: Migrate ufw rules to Ansible's ufw module or firewalld for RHEL-based systems.
- **fail2ban Configuration**: Migrate to Ansible's template module for fail2ban configuration.
- **SSH Hardening**: Maintain SSH security configurations (disable root login, password authentication).
- **Redis Authentication**: Ensure Redis password is securely managed in Ansible Vault.
- **PostgreSQL Authentication**: Secure database credentials using Ansible Vault.

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of site configurations based on node attributes will need to be replicated using Ansible's templating system.
- **SSL Certificate Generation**: Self-signed certificate generation logic needs to be migrated to Ansible's crypto modules.
- **Security Hardening**: Comprehensive security measures need careful migration to maintain the same level of protection.
- **Database Initialization**: PostgreSQL database and user creation needs to be handled idempotently.
- **Service Orchestration**: Ensuring proper service dependencies and restart handlers are maintained.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Base Nginx installation
   - SSL certificate generation
   - Site configuration templates
   - Security hardening (separate task)

2. **cache** (moderate complexity)
   - Memcached configuration
   - Redis installation and configuration

3. **fastapi-tutorial** (high complexity)
   - PostgreSQL installation and configuration
   - Python environment setup
   - Application deployment
   - Service configuration

4. **Security Hardening** (high complexity, cross-cutting)
   - Firewall rules
   - fail2ban configuration
   - System hardening

### Assumptions

1. The target environment will continue to be Fedora-based systems, with potential support for Ubuntu and CentOS as mentioned in the cookbook metadata.
2. Self-signed certificates are acceptable for development, but production deployment may require proper CA-signed certificates.
3. The security configurations are appropriate for the target environment and don't need significant changes beyond the migration.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git is accessible and contains the expected code.
5. The Redis configuration hack in the cache cookbook is addressing a specific issue that may need investigation during migration.
6. The current Chef setup is functional and represents the desired end state for the Ansible migration.
7. No containerization is currently in use or planned as part of this migration.
8. No CI/CD pipeline integration is explicitly defined in the current setup.