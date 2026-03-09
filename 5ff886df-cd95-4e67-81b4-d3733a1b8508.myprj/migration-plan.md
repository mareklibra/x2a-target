# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- **Total**: 7 weeks

**Complexity**: Medium to High
- Multiple interconnected services
- Security configurations
- SSL certificate management
- Database integration

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

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `Policyfile.rb`: Chef policy file defining the run list and cookbook dependencies
- `Vagrantfile`: Defines the development VM configuration using Vagrant
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef
- `solo.json`: Configuration data for Chef solo, contains site configurations and security settings
- `solo.rb`: Chef solo configuration file

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) as specified in cookbook metadata files. The Vagrantfile uses Fedora 42.
- **Virtual Machine Technology**: Vagrant with libvirt provider as specified in the Vagrantfile
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud VMs

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks
- **ssl_certificate (~> 2.1)**: Replace with Ansible's openssl_* modules for certificate management

### Security Considerations

- **SSL Certificate Management**: Migration must handle self-signed certificate generation for development environments
- **Firewall Configuration**: UFW configuration must be migrated to equivalent Ansible ufw module
- **Fail2ban Setup**: Fail2ban configuration must be preserved in Ansible tasks
- **SSH Hardening**: SSH security settings (disabling root login, password authentication) must be maintained
- **Redis Authentication**: Redis password must be securely managed in Ansible Vault

### Technical Challenges

- **Multi-site Configuration**: The dynamic generation of multiple Nginx site configurations must be preserved
- **SSL Certificate Management**: Self-signed certificate generation and management must be handled properly
- **Service Dependencies**: Ensure proper ordering of service installation and configuration
- **Idempotency**: Ensure all operations remain idempotent, especially database user creation
- **Password Management**: Secure handling of Redis and PostgreSQL passwords

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Base Nginx installation
   - SSL certificate management
   - Site configuration
   - Security hardening

2. **cache** (low complexity, standalone service)
   - Memcached configuration
   - Redis installation and security

3. **fastapi-tutorial** (high complexity, depends on database)
   - PostgreSQL installation and configuration
   - Python environment setup
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Vagrant VMs for development/testing
2. Self-signed certificates are acceptable for development environments
3. The same security policies should be maintained in the Ansible implementation
4. The FastAPI application source will continue to be pulled from the same Git repository
5. Redis and PostgreSQL passwords will need to be managed securely in Ansible Vault
6. The same operating system support (Ubuntu 18.04+, CentOS 7+) is required

## Ansible Implementation Plan

### Directory Structure

```
ansible/
├── inventory/
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── webservers.yml
│   └── hosts.ini
├── roles/
│   ├── nginx_multisite/
│   ├── cache_services/
│   └── fastapi_app/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
└── ansible.cfg
```

### Key Implementation Tasks

1. **Create Base Ansible Configuration**
   - Set up inventory structure
   - Configure ansible.cfg
   - Create Vagrant test environment

2. **Develop nginx_multisite Role**
   - Implement Nginx installation and configuration
   - Create SSL certificate management tasks
   - Implement site configuration templates
   - Migrate security hardening (fail2ban, ufw)

3. **Develop cache_services Role**
   - Implement Memcached installation and configuration
   - Implement Redis installation with authentication
   - Create service management tasks

4. **Develop fastapi_app Role**
   - Implement PostgreSQL installation and configuration
   - Create Python environment setup
   - Implement application deployment from Git
   - Configure systemd service

5. **Create Ansible Vault for Secrets**
   - Store Redis password
   - Store PostgreSQL credentials

6. **Create Main Playbook**
   - Combine all roles with proper ordering
   - Implement tags for selective execution

7. **Testing and Validation**
   - Test on Vagrant VMs
   - Verify all services are running correctly
   - Validate security configurations

8. **Documentation**
   - Document role usage and variables
   - Create migration completion report