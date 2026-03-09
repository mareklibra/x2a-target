# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated timeline for migration is 3-4 weeks, with moderate complexity due to the security configurations and multi-site SSL setup.

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

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `Policyfile.rb`: Chef policy file defining the run list and cookbook dependencies
- `solo.rb`: Chef Solo configuration file specifying cookbook paths and log settings
- `solo.json`: Node attributes and run list for Chef Solo
- `Vagrantfile`: Defines a Fedora 42 VM for local development and testing
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` module or community.general collection
- **memcached (~> 6.0)**: Replace with Ansible's `memcached` module or dedicated role
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role or tasks using the `community.general.redis` module
- **ssl_certificate (~> 2.1)**: Replace with Ansible's `openssl_*` modules for certificate management

### Security Considerations

- **SSL Certificate Management**: Migration must preserve the self-signed certificate generation for development environments
- **Firewall Configuration**: UFW rules need to be migrated to equivalent Ansible UFW module tasks
- **fail2ban Setup**: Configuration needs to be migrated to Ansible tasks
- **SSH Hardening**: SSH configuration hardening (disabling root login, password authentication) needs to be preserved
- **Redis Authentication**: Redis password must be securely managed in Ansible Vault

### Technical Challenges

- **Multi-site Configuration**: The dynamic generation of multiple Nginx site configurations needs careful translation to Ansible templates
- **SSL Certificate Management**: Self-signed certificate generation logic needs to be preserved
- **Security Hardening**: Comprehensive security measures need to be maintained across the migration
- **Service Dependencies**: Proper ordering of service installations and configurations must be maintained

### Migration Order

1. **fastapi-tutorial** (moderate complexity, standalone application)
   - Python package installation
   - PostgreSQL database setup
   - Application deployment
   - Systemd service configuration

2. **cache** (moderate complexity, service configuration)
   - Memcached installation and configuration
   - Redis installation with authentication
   - Service management

3. **nginx-multisite** (high complexity, depends on other services)
   - Base Nginx installation
   - SSL certificate management
   - Multi-site configuration
   - Security hardening

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions (Ubuntu 18.04+, CentOS 7+)
2. Self-signed certificates are acceptable for development environments
3. The same security hardening measures are required in the Ansible implementation
4. The FastAPI application source will continue to be pulled from the same Git repository
5. Redis will continue to require password authentication
6. The same Nginx site configurations will be maintained
7. No additional monitoring or logging requirements beyond what's in the current Chef implementation

## Implementation Details

### Ansible Structure

```
ansible/
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml
│       └── webservers.yml
├── roles/
│   ├── nginx-multisite/
│   ├── cache/
│   └── fastapi-tutorial/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
└── ansible.cfg
```

### Variable Migration

Chef node attributes will be converted to Ansible variables in:
- `group_vars/all.yml` for global settings
- `group_vars/webservers.yml` for web server specific settings
- Role defaults for role-specific variables

### Secrets Management

- Redis password and database credentials should be stored in Ansible Vault
- SSL private keys should be handled securely using Ansible Vault or external secret management

### Testing Strategy

1. Develop individual roles with molecule tests
2. Create a Vagrant-based test environment similar to the existing one
3. Test the full playbook against the Vagrant environment
4. Compare the results with the current Chef implementation

## Timeline Estimate

- **Week 1**: Analysis and planning, role structure setup
- **Week 2**: Implement fastapi-tutorial and cache roles
- **Week 3**: Implement nginx-multisite role
- **Week 4**: Integration testing, documentation, and knowledge transfer

Total estimated effort: 3-4 weeks for a single developer, depending on familiarity with both Chef and Ansible.