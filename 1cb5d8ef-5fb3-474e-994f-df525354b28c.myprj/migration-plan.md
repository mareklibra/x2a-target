# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for a multi-site Nginx web server with caching capabilities (Memcached and Redis). The migration scope is moderate, involving two primary cookbooks with external dependencies. The estimated timeline for migration is 2-3 weeks for a single engineer, or 1 week with a team of 2-3 engineers working in parallel.

The repository uses Chef Solo with Berkshelf for dependency management and is designed to run in a Vagrant-based development environment using Ubuntu 22.04. The infrastructure focuses on serving multiple SSL-enabled websites with security hardening and caching capabilities.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall)

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

### Infrastructure Files

- `Berksfile`: Dependency management file listing cookbook dependencies (nginx, memcached, redisio)
- `Policyfile.rb`: Chef policy file defining the run list and cookbook dependencies
- `Policyfile.lock.json`: Locked versions of cookbook dependencies
- `solo.json`: Configuration data for Chef Solo, including site configurations and security settings
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Defines the development environment using Ubuntu 22.04 with libvirt provider
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Ubuntu 22.04 LTS (specified in Vagrantfile)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or local development

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration
- **ssl_certificate (~> 2.1)**: Replace with Ansible's openssl_* modules for certificate management

### Security Considerations

- **Firewall (ufw)**: Migrate to Ansible's `ufw` module for firewall management
- **Fail2ban**: Migrate to Ansible tasks for fail2ban installation and configuration
- **SSH hardening**: Migrate SSH security configurations using Ansible's `lineinfile` or templates
- **SSL/TLS**: Migrate self-signed certificate generation using Ansible's `openssl_*` modules
- **Redis authentication**: Ensure Redis password is stored securely in Ansible Vault

### Technical Challenges

- **Multi-site configuration**: Ensure the dynamic generation of multiple Nginx virtual hosts is properly implemented in Ansible using loops and templates
- **SSL certificate management**: Implement proper certificate generation and management in Ansible
- **Security hardening**: Ensure all security measures are properly translated to Ansible equivalents
- **Redis configuration hack**: The Chef recipe contains a hack to fix Redis configuration files; ensure this is properly addressed in Ansible

### Migration Order

1. **cache cookbook** (moderate complexity)
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Address Redis configuration hack

2. **nginx-multisite cookbook** (higher complexity)
   - Implement base Nginx configuration
   - Implement security hardening (fail2ban, ufw, sysctl)
   - Implement SSL certificate generation
   - Implement multi-site configuration

### Assumptions

1. The target environment will continue to be Ubuntu 22.04 LTS
2. Self-signed certificates are acceptable for development purposes
3. The current security configurations are appropriate for the target environment
4. The Redis password in the Chef recipe is a placeholder and will be replaced with a secure password in Ansible Vault
5. The Nginx sites configuration in solo.json will be migrated to Ansible inventory variables
6. The migration will maintain the same functionality but follow Ansible best practices
7. The development workflow will continue to use Vagrant for testing