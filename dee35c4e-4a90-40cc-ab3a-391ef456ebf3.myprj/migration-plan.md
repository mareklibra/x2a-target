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
- `solo.json`: Node attributes and run list for Chef Solo execution
- `Vagrantfile`: Defines a Fedora 42 VM for testing with port forwarding and networking
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks
- **ssl_certificate (~> 2.1)**: Replace with Ansible openssl_* modules for certificate management

### Security Considerations

- **SSL Certificate Management**: Migration must handle self-signed certificate generation for multiple sites
- **Firewall Configuration**: UFW rules need to be converted to appropriate firewall module (firewalld for Fedora)
- **fail2ban Configuration**: Ensure fail2ban setup is properly migrated with equivalent templates
- **SSH Hardening**: Preserve SSH security settings (root login disabled, password authentication disabled)
- **Redis Authentication**: Ensure Redis password is securely managed in Ansible Vault

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of multiple site configurations with SSL needs careful planning
- **Service Dependencies**: Ensure proper ordering of service installations and configurations (e.g., PostgreSQL before FastAPI app)
- **Python Environment Management**: Converting the Python virtual environment setup to idempotent Ansible tasks
- **Security Hardening**: Ensuring all security measures are properly implemented in Ansible

### Migration Order

1. **cache cookbook** (low complexity, foundational service)
   - Implement Memcached configuration
   - Implement Redis with authentication

2. **nginx-multisite cookbook** (moderate complexity, core infrastructure)
   - Implement base Nginx configuration
   - Implement SSL certificate generation
   - Implement security hardening (fail2ban, firewall)
   - Implement multi-site configuration

3. **fastapi-tutorial cookbook** (high complexity, application layer)
   - Implement PostgreSQL database setup
   - Implement Python environment and application deployment
   - Implement systemd service configuration

### Assumptions

1. The target environment will continue to be Fedora 42 or similar Linux distributions
2. Self-signed certificates are acceptable for the migrated solution (not using Let's Encrypt or other CA)
3. The same security posture (fail2ban, firewall, SSH hardening) is required in the Ansible implementation
4. The FastAPI application source will continue to be pulled from the same Git repository
5. Redis password and PostgreSQL credentials will need to be stored securely in Ansible Vault
6. The same site configurations (test.cluster.local, ci.cluster.local, status.cluster.local) will be maintained
7. The Vagrant testing environment will be preserved but updated to use Ansible provisioning