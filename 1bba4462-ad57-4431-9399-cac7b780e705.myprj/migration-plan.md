# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for deploying a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI Python application with PostgreSQL. The migration to Ansible will involve converting Chef cookbooks, recipes, templates, and attributes to Ansible roles, playbooks, templates, and variables.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- Multiple interconnected services
- Security configurations that need careful migration
- Database and application deployment with specific requirements

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL/TLS setup with self-signed certificates, security hardening (fail2ban, ufw firewall), custom Nginx configurations

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx, ssl_certificate, memcached, redisio) - will be replaced by Ansible Galaxy requirements
- `Policyfile.rb`: Defines the run list and cookbook dependencies - will be replaced by Ansible playbooks
- `Vagrantfile`: Defines the development VM configuration - can be adapted for Ansible testing
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `solo.json`: Chef attributes and run list - will be converted to Ansible variables
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM - will be replaced by Ansible provisioning

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (from Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Libvirt (from Vagrantfile)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management tasks or community roles

### Security Considerations

- **SSL/TLS Configuration**: Migrate self-signed certificate generation and configuration, ensuring proper file permissions
- **Firewall (ufw)**: Migrate firewall rules to Ansible ufw module or firewalld for Fedora/RHEL systems
- **fail2ban**: Migrate fail2ban configuration to Ansible fail2ban role or direct configuration
- **SSH Hardening**: Migrate SSH security configurations (disable root login, password authentication)
- **System Hardening**: Migrate sysctl security configurations
- **Nginx Security Headers**: Ensure all security headers are properly configured in Ansible templates
- **Redis Authentication**: Securely manage Redis password in Ansible Vault
- **PostgreSQL Authentication**: Securely manage database credentials in Ansible Vault

### Technical Challenges

- **Multi-site Configuration**: Ensure the dynamic generation of multiple Nginx site configurations is properly implemented in Ansible
- **SSL Certificate Management**: Implement proper certificate generation and management in Ansible
- **Service Dependencies**: Maintain proper ordering of service installation and configuration
- **Database Initialization**: Ensure PostgreSQL database creation and user setup is properly handled
- **Python Environment**: Properly manage Python virtual environment and dependencies
- **Service Management**: Ensure proper systemd service configuration and management

### Migration Order

1. **Base Infrastructure** (low complexity)
   - System packages
   - Basic system configuration
   - Firewall and security basics

2. **Nginx Configuration** (medium complexity)
   - Base Nginx installation
   - SSL certificate generation
   - Site configuration templates
   - Security hardening

3. **Caching Services** (medium complexity)
   - Memcached configuration
   - Redis installation and security

4. **Application Deployment** (high complexity)
   - PostgreSQL database setup
   - Python environment configuration
   - FastAPI application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora/RHEL-based systems, though the cookbooks support Ubuntu as well
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or other CA)
3. The security requirements will remain the same (fail2ban, firewall, SSH hardening)
4. The FastAPI application source will continue to be pulled from the same Git repository
5. The multi-site configuration with three virtual hosts will be maintained
6. Redis will continue to require password authentication
7. The PostgreSQL database structure and user permissions will remain the same
8. The systemd service configuration for the FastAPI application will remain similar
9. The Vagrant development environment will be maintained for testing