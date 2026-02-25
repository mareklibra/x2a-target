# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for a multi-site Nginx setup with caching services (Memcached and Redis) and a FastAPI application with PostgreSQL. The migration to Ansible is estimated to be of medium complexity, requiring approximately 3-4 weeks of effort for a complete migration with testing. The repository consists of 3 cookbooks with clear responsibilities and dependencies on external cookbooks from the Chef Supermarket.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and self-signed certificates
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate management, security hardening with fail2ban and UFW

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git-based deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management file listing cookbook dependencies (nginx, memcached, redisio)
- `Policyfile.rb`: Chef Policyfile defining the run list and cookbook dependencies
- `Policyfile.lock.json`: Locked versions of cookbook dependencies
- `solo.json`: Node attributes for Chef Solo, including Nginx site configurations and security settings
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Defines a Fedora 42 VM for testing with port forwarding and networking
- `vagrant-provision.sh`: Bash script to provision the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct configuration
- **ssl_certificate (~> 2.1)**: Replace with Ansible openssl modules for certificate generation

### Security Considerations

- **fail2ban configuration**: Migrate fail2ban jail configuration to Ansible templates
- **UFW firewall rules**: Use Ansible UFW module to configure firewall rules
- **SSH hardening**: Migrate SSH security configurations using Ansible's openssh_config module
- **SSL certificate management**: Use Ansible's openssl_* modules for certificate generation
- **Redis password**: Store Redis password in Ansible Vault
- **PostgreSQL credentials**: Store database credentials in Ansible Vault

### Technical Challenges

- **Multi-site Nginx configuration**: Ensure proper templating of multiple virtual hosts with conditional SSL
- **Redis configuration hack**: The Chef cookbook uses a ruby_block to modify Redis config files directly; this needs a cleaner approach in Ansible
- **Service dependencies**: Ensure proper ordering of service installations and configurations, particularly for the FastAPI application which depends on PostgreSQL

### Migration Order

1. **nginx-multisite cookbook** (medium complexity)
   - Base Nginx installation and configuration
   - Security hardening (fail2ban, UFW)
   - SSL certificate generation
   - Virtual host configuration

2. **cache cookbook** (low complexity)
   - Memcached installation and configuration
   - Redis installation and configuration

3. **fastapi-tutorial cookbook** (high complexity)
   - PostgreSQL installation and database setup
   - Python environment setup
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions.
2. Self-signed certificates are acceptable for development/testing; production would require proper certificates.
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
4. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment.
5. Redis and Memcached configurations don't require clustering or high availability.
6. The current directory structure in `/opt/server/` for website content and `/opt/fastapi-tutorial/` for the application will be maintained.
7. The PostgreSQL database will continue to run on the same host as the application.
8. The current systemd service configuration for the FastAPI application is appropriate.
9. The Nginx sites configuration with test.cluster.local, ci.cluster.local, and status.cluster.local domains will be maintained.