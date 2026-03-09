# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Memcached and Redis) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has well-structured Chef cookbooks with clear dependencies
- Security configurations need careful migration
- Multiple services with interdependencies require coordinated testing

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall)

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
- `Policyfile.rb`: Chef Policyfile defining the run list and cookbook dependencies
- `solo.json`: Configuration data for Chef Solo, contains site configurations and security settings
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Defines a Fedora 42 VM for development/testing with port forwarding and networking
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (18.04+) and CentOS (7.0+), with Fedora 42 used for development
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role (e.g., geerlingguy.nginx)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., geerlingguy.memcached)
- **redisio (~> 7.2.4)**: Replace with Ansible redis role (e.g., geerlingguy.redis)
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management modules (openssl_certificate, openssl_privatekey)

### Security Considerations

- **SSL Certificate Management**: Migration must handle self-signed certificate generation for development environments
- **Firewall Configuration**: UFW firewall rules need to be migrated to equivalent Ansible ufw module tasks
- **Fail2ban Configuration**: Fail2ban setup needs to be migrated to Ansible tasks
- **SSH Hardening**: SSH security configurations (disable root login, password authentication) need careful migration
- **Redis Authentication**: Redis password must be securely managed in Ansible Vault
- **PostgreSQL Credentials**: Database credentials should be stored in Ansible Vault

### Technical Challenges

- **Multi-site Configuration**: The dynamic generation of multiple Nginx site configurations needs to be replicated in Ansible using loops and templates
- **Service Coordination**: Proper ordering of service installation, configuration, and startup needs to be maintained
- **SSL Certificate Generation**: Self-signed certificate generation logic needs to be replicated
- **Python Environment Management**: Python virtual environment setup and dependency installation needs careful migration
- **PostgreSQL User/Database Creation**: Database initialization and user creation needs to be handled with idempotent Ansible tasks

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Base Nginx installation and configuration
   - SSL certificate management
   - Virtual host configuration
   - Security hardening (fail2ban, ufw)

2. **cache** (low complexity, independent service)
   - Memcached configuration
   - Redis installation and security configuration

3. **fastapi-tutorial** (high complexity, depends on database)
   - PostgreSQL installation and configuration
   - Python environment setup
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be either Ubuntu 18.04+ or CentOS 7.0+
2. The Vagrant development environment will be maintained but converted to use Ansible provisioner
3. Self-signed certificates are acceptable for development environments
4. The FastAPI application source will continue to be pulled from the same Git repository
5. The security requirements (fail2ban, ufw, SSH hardening) will remain the same
6. Redis will continue to require password authentication
7. The PostgreSQL database structure and user permissions will remain unchanged
8. The systemd service configuration for FastAPI will remain similar
9. The Nginx virtual host structure with multiple sites will be maintained
10. The current directory structure in the target environment (/var/www/, /opt/fastapi-tutorial) will be preserved