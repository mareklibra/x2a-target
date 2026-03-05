# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for deploying a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI Python application with PostgreSQL. The migration to Ansible will involve converting Chef cookbooks, recipes, templates, and attributes to Ansible roles, playbooks, templates, and variables.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1 week
- Documentation and Knowledge Transfer: 1 week
- Total: 5-6 weeks

**Complexity:** Medium - The repository has well-structured Chef cookbooks with clear dependencies and configuration patterns that can be mapped to Ansible roles.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw, sysctl)

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment, Git repository deployment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (both local and external) - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Defines Chef policy with run list and cookbook dependencies - will be replaced by Ansible playbook structure
- `Vagrantfile`: Defines development VM configuration - can be adapted for Ansible testing
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `solo.json`: Chef attributes and run list - will be replaced by Ansible inventory and variables
- `vagrant-provision.sh`: Shell script for provisioning Vagrant VM - will be replaced by Ansible provisioning

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or local development environments

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management tasks or community role

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migrate to Ansible's crypto modules or community roles for certificate management.
- **Firewall Configuration (ufw)**: Replace with Ansible's firewall modules (ufw or firewalld depending on target OS).
- **fail2ban Configuration**: Migrate fail2ban configuration to Ansible tasks or community role.
- **SSH Hardening**: Migrate SSH security configurations to Ansible's openssh_config module.
- **System Hardening (sysctl)**: Use Ansible's sysctl module to apply the same security configurations.
- **Redis Authentication**: Ensure Redis password is stored securely using Ansible Vault.
- **PostgreSQL Authentication**: Ensure database credentials are stored securely using Ansible Vault.

### Technical Challenges

- **Multi-site Nginx Configuration**: Ensure the dynamic generation of site configurations is properly implemented in Ansible templates.
- **SSL Certificate Generation**: Implement proper certificate management with Ansible crypto modules.
- **Service Dependencies**: Maintain proper ordering of service installations and configurations.
- **Security Hardening**: Ensure all security measures are properly implemented in Ansible.
- **Database User and Schema Creation**: Ensure idempotent PostgreSQL user and database creation.

### Migration Order

1. **Base Infrastructure** (low complexity)
   - Basic system configuration
   - Package installations
   - Directory structure

2. **Security Components** (medium complexity)
   - Firewall configuration
   - fail2ban setup
   - SSH hardening
   - System hardening (sysctl)

3. **Nginx Configuration** (medium complexity)
   - Base Nginx installation and configuration
   - SSL certificate generation
   - Site configuration templates

4. **Caching Services** (medium complexity)
   - Memcached installation and configuration
   - Redis installation and configuration with authentication

5. **FastAPI Application** (high complexity)
   - PostgreSQL installation and configuration
   - Database and user creation
   - Python environment setup
   - Application deployment
   - Systemd service configuration

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as indicated in the Vagrantfile).
2. Self-signed SSL certificates are acceptable for the migrated solution (production environments might require proper CA-signed certificates).
3. The security requirements (fail2ban, ufw, SSH hardening) will remain the same in the Ansible implementation.
4. The FastAPI application source will continue to be pulled from the same Git repository.
5. The multi-site configuration pattern with three sites (test.cluster.local, ci.cluster.local, status.cluster.local) will be maintained.
6. Redis and Memcached will continue to be used as caching solutions.
7. The current directory structure and naming conventions can be adapted to Ansible best practices.
8. The PostgreSQL database configuration for the FastAPI application will remain similar.
9. The current security credentials in the Chef recipes (Redis password, PostgreSQL credentials) are for development only and will be replaced with proper secret management in Ansible.