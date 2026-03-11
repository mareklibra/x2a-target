# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks to Ansible roles and playbooks, addressing external dependencies, and ensuring security configurations are properly maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has well-structured Chef cookbooks with clear dependencies
- Security configurations need careful migration
- Multiple services with interdependencies (Nginx, Redis, Memcached, PostgreSQL, FastAPI)

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
    - Key Features: Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (both local and external from Chef Supermarket)
- `Policyfile.rb`: Defines the Chef policy with run list and cookbook dependencies
- `Vagrantfile`: Defines the development VM configuration using Fedora 42
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef
- `solo.json`: Configuration data for Chef Solo, including site configurations and security settings
- `solo.rb`: Chef Solo configuration file

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) as indicated in cookbook metadata files. Development environment uses Fedora 42 (from Vagrantfile).
- **Virtual Machine Technology**: Vagrant with libvirt provider (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` module and templates
- **memcached (~> 6.0)**: Use Ansible's `apt`/`yum` modules to install and configure memcached
- **redisio (~> 7.2.4)**: Use Ansible's `apt`/`yum` modules to install Redis and template for configuration
- **ssl_certificate (~> 2.1)**: Use Ansible's `openssl_*` modules for certificate management

### Security Considerations

- **SSL Certificate Management**: Migrate self-signed certificate generation to Ansible's `openssl_certificate` module
- **Firewall Configuration (UFW)**: Use Ansible's `ufw` module to configure firewall rules
- **fail2ban Configuration**: Use Ansible's package management and templates to configure fail2ban
- **SSH Hardening**: Ensure SSH security configurations are maintained using Ansible's `lineinfile` or templates
- **Redis Password**: Store Redis password in Ansible Vault instead of plaintext in recipes

### Technical Challenges

- **Multi-site Nginx Configuration**: Ensure the dynamic generation of site configurations is properly migrated to Ansible templates and loops
- **SSL Certificate Management**: Properly handle certificate generation and permissions in Ansible
- **Service Dependencies**: Maintain proper ordering of service installations and configurations
- **PostgreSQL User/Database Creation**: Ensure idempotent database operations using Ansible's PostgreSQL modules

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for web services)
   - Create base Nginx role
   - Implement SSL certificate management
   - Configure multi-site setup
   - Implement security hardening

2. **cache** (low complexity, standalone services)
   - Create Memcached role
   - Create Redis role with authentication

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - Create PostgreSQL role
   - Create Python application deployment role
   - Configure systemd service

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS as specified in the cookbook metadata
2. Self-signed certificates are acceptable for development; production may require integration with Let's Encrypt or other certificate providers
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain accessible
4. The Redis password "redis_secure_password_123" is for development and will be replaced with a secure password in production
5. The PostgreSQL credentials (user: fastapi, password: fastapi_password) are for development and will be replaced with secure credentials in production
6. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment

## Ansible Structure Recommendation

```
ansible-project/
├── inventories/
│   ├── development/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
├── roles/
│   ├── nginx-multisite/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── tasks/
│   │   └── templates/
│   ├── cache/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── tasks/
│   │   └── templates/
│   └── fastapi-tutorial/
│       ├── defaults/
│       ├── handlers/
│       ├── tasks/
│       └── templates/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
└── group_vars/
    └── all/
        ├── vars.yml
        └── vault.yml
```

## Testing Strategy

1. Create Vagrant-based test environment similar to the current setup
2. Develop Ansible playbooks incrementally, testing each component
3. Verify functionality matches the original Chef implementation:
   - Nginx sites are properly configured with SSL
   - Security hardening is applied
   - Caching services are operational
   - FastAPI application is deployed and functional
4. Perform integration testing to ensure all components work together

## Knowledge Transfer Plan

1. Document each Ansible role with README files explaining purpose and configuration options
2. Create example inventory files for different environments
3. Provide a migration summary document highlighting key differences between the Chef and Ansible implementations
4. Conduct knowledge transfer sessions with the team