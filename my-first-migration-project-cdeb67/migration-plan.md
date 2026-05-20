# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application with PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies on community cookbooks need to be replaced with Ansible equivalents
- Security configurations and SSL certificate management require careful handling
- Secrets management needs to be implemented with Ansible Vault

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, UFW firewall)

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints. Will be replaced by Ansible requirements.yml.
- `solo.json`: Chef configuration file containing the run list and node attributes. Will be replaced by Ansible inventory variables.
- `solo.rb`: Chef configuration file specifying cookbook paths and log settings. Will be replaced by Ansible configuration.
- `Vagrantfile`: Defines the development VM configuration using Vagrant. Can be adapted for Ansible by changing the provisioner.
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef. Will be replaced with Ansible provisioning.

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (from Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Libvirt (from Vagrantfile)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration

### Security Considerations

- **SSL Certificate Management**: 
  - Self-signed certificates are generated for each site
  - Migration approach: Use Ansible's openssl_* modules or community.crypto collection

- **Firewall Configuration**: 
  - UFW firewall is configured with specific rules
  - Migration approach: Use Ansible's ufw module or firewalld module depending on target OS

- **Fail2ban Configuration**: 
  - Fail2ban is installed and configured for intrusion prevention
  - Migration approach: Use Ansible to install and configure fail2ban with templates

- **SSH Hardening**: 
  - Root login disabled and password authentication disabled
  - Migration approach: Use Ansible's lineinfile or template module to configure SSH

- **Vault/secrets management**:
  - Redis password is hardcoded in the Chef recipe
  - PostgreSQL credentials are hardcoded in the FastAPI recipe
  - Migration approach: Move all credentials to Ansible Vault

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation dynamically generates site configurations based on node attributes
  - Mitigation: Create Ansible templates with Jinja2 loops to generate similar configurations from host variables

- **SSL Certificate Generation**: 
  - Description: Self-signed certificates are generated for each site
  - Mitigation: Use Ansible's openssl_* modules to generate certificates or integrate with Let's Encrypt using community modules

- **Redis Configuration Hack**: 
  - Description: The current implementation includes a ruby_block to modify Redis configuration files after they're created
  - Mitigation: Create a custom Redis configuration template in Ansible that doesn't require post-processing

- **PostgreSQL User and Database Creation**: 
  - Description: Uses shell commands to create database users and permissions
  - Mitigation: Use Ansible's postgresql_* modules for more idempotent database management

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Create base Nginx role
   - Implement SSL certificate generation
   - Configure virtual hosts
   - Implement security hardening

2. **cache** (moderate complexity, depends on network configuration)
   - Create Memcached role
   - Create Redis role with authentication
   - Ensure proper integration with Nginx

3. **fastapi-tutorial** (highest complexity, depends on other components)
   - Create PostgreSQL role
   - Implement Python application deployment
   - Configure systemd service
   - Set up environment variables

### Assumptions

1. The target environment will continue to use Fedora or a similar Linux distribution.
2. Self-signed certificates are acceptable for the migrated environment, or proper certificates will be provided.
3. The same security requirements (fail2ban, UFW, SSH hardening) apply to the target environment.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
5. The current VM networking configuration (ports, IP addresses) should be preserved.
6. No custom Chef resources or libraries are used beyond what's visible in the examined files.
7. Redis and Memcached configurations don't have specific tuning requirements beyond what's in the recipes.
8. The PostgreSQL database schema is managed by the FastAPI application, not by the infrastructure code.