# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Memcached and Redis) and a FastAPI application with PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:**
- Medium complexity due to multiple services and security configurations
- Straightforward conversion patterns with well-defined Chef cookbooks
- Some security considerations with SSL certificates and passwords

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw), custom nginx configurations

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Memcached configuration, Redis with password authentication, service management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints. Will be replaced by Ansible Galaxy requirements.yml.
- `solo.json`: Contains Chef run list and node attributes. Will be converted to Ansible group_vars or host_vars.
- `solo.rb`: Chef Solo configuration file. Will be replaced by Ansible configuration.
- `Vagrantfile`: Defines the development VM configuration. Can be adapted for Ansible testing with minimal changes.
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks. Will be replaced with Ansible provisioning.

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ mentioned in cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
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
  - UFW is configured with specific rules
  - Migration approach: Use Ansible's ufw module or firewalld module depending on target OS

- **Fail2ban Configuration**: 
  - Fail2ban is installed and configured
  - Migration approach: Use Ansible to install and configure fail2ban with templates

- **SSH Hardening**: 
  - Root login disabled
  - Password authentication disabled
  - Migration approach: Use Ansible's lineinfile or template module to configure SSH

- **Vault/secrets management**:
  - Redis password hardcoded in recipe: "redis_secure_password_123"
  - PostgreSQL password hardcoded in recipe: "fastapi_password"
  - Migration approach: Use Ansible Vault to secure these credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current setup dynamically generates Nginx site configurations based on node attributes
  - Mitigation: Use Ansible templates with loops over site definitions in variables

- **SSL Certificate Generation**: 
  - Description: Self-signed certificates are generated for each site
  - Mitigation: Use Ansible's openssl_* modules to generate certificates or integrate with Let's Encrypt

- **Service Dependencies**: 
  - Description: Proper ordering of service installation and configuration
  - Mitigation: Use Ansible handlers and proper task ordering to ensure services are configured correctly

- **Database User and Schema Creation**: 
  - Description: PostgreSQL user and database creation with proper permissions
  - Mitigation: Use Ansible's postgresql_* modules to manage database objects

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Base Nginx installation and configuration
   - Security hardening (fail2ban, ufw, sysctl)
   - SSL certificate generation
   - Virtual host configuration

2. **cache** (low complexity, independent service)
   - Memcached installation and configuration
   - Redis installation and configuration with authentication

3. **fastapi-tutorial** (high complexity, depends on database)
   - PostgreSQL installation and configuration
   - Python environment setup
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora-based, but with compatibility for Ubuntu and CentOS as mentioned in the cookbook metadata.
2. Self-signed certificates are acceptable for the migrated solution, but production environments might require proper certificates.
3. The security configurations (fail2ban, ufw, SSH hardening) are required in the migrated solution.
4. The FastAPI application source code will continue to be pulled from the same Git repository.
5. The current Redis and PostgreSQL passwords are development passwords and should be replaced with proper secrets management in production.
6. The Vagrant development environment should be preserved for testing the Ansible playbooks.
7. No specific monitoring or logging solutions are mentioned in the current configuration, so they are not included in the migration plan.
8. The current configuration does not include backup strategies for Redis or PostgreSQL data.