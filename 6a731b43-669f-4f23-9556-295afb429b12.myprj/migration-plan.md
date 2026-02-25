# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI Python application with PostgreSQL. The migration to Ansible is estimated to be of medium complexity, requiring approximately 3-4 weeks of effort with 1-2 engineers. The primary challenges include handling SSL certificate management, security configurations, and ensuring proper service dependencies.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Multi-site Nginx configuration with SSL support, security hardening, and site-specific configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: SSL certificate generation, fail2ban integration, UFW firewall configuration, multiple virtual hosts

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies including nginx (~> 12.0), memcached (~> 6.0), and redisio (~> 7.2.4)
- `Policyfile.rb`: Defines the run list and cookbook dependencies for Chef Policyfile workflow
- `Vagrantfile`: Configures a Fedora 42 VM with port forwarding for development and testing
- `solo.json`: Contains node attributes including Nginx site configurations and security settings
- `solo.rb`: Chef Solo configuration file defining cookbook paths and log settings
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration) with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role from Galaxy or custom role
- **memcached (~> 6.0)**: Replace with Ansible memcached role from Galaxy or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role from Galaxy or custom role with authentication support
- **ssl_certificate (~> 2.1)**: Replace with Ansible's openssl_* modules for certificate management

### Security Considerations

- **fail2ban configuration**: Migrate fail2ban jail configuration to Ansible template
- **UFW firewall rules**: Use Ansible's ufw module to configure firewall rules
- **SSH hardening**: Implement SSH configuration using Ansible's lineinfile or template module
- **SSL/TLS configuration**: Ensure proper SSL certificate generation and configuration
- **Redis password**: Store Redis password in Ansible Vault for secure management
- **PostgreSQL credentials**: Store database credentials in Ansible Vault

### Technical Challenges

- **Self-signed certificate generation**: Implement using Ansible's openssl_* modules
- **Multi-site configuration**: Create a flexible Ansible role for managing multiple Nginx sites
- **Custom resource migration**: Convert the Chef `lineinfile` custom resource to Ansible's native lineinfile module
- **Service dependencies**: Ensure proper ordering of service installation and configuration
- **PostgreSQL user/database creation**: Implement using Ansible's postgresql_* modules

### Migration Order

1. **nginx-multisite cookbook** (moderate complexity)
   - Begin with basic Nginx installation and configuration
   - Implement security hardening (fail2ban, UFW)
   - Add SSL certificate generation
   - Configure virtual hosts

2. **cache cookbook** (low complexity)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial cookbook** (moderate complexity)
   - Set up PostgreSQL database
   - Deploy FastAPI application
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions
2. Self-signed certificates are acceptable for development/testing (production would likely use Let's Encrypt or other CA)
3. The same directory structure for web content will be maintained
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. The security requirements (fail2ban, UFW, SSH hardening) will remain the same
6. Redis will continue to require password authentication
7. The PostgreSQL database structure and user permissions will remain unchanged

## Detailed Migration Tasks

### 1. Infrastructure Setup

- Create Ansible directory structure following best practices
- Set up inventory file with appropriate groups
- Create group_vars and host_vars directories for variable management
- Set up Ansible Vault for sensitive information (Redis password, PostgreSQL credentials)

### 2. Nginx Multi-site Role

- Create role structure with tasks, templates, files, and handlers
- Migrate nginx.conf.erb to Jinja2 template
- Migrate site.conf.erb to Jinja2 template with support for multiple sites
- Implement SSL certificate generation using openssl_* modules
- Migrate security configurations (fail2ban, UFW, sysctl)
- Copy static website content from files directory

### 3. Cache Services Role

- Create role for Memcached installation and configuration
- Create role for Redis installation with authentication
- Implement log directory management
- Set up service handlers for restarts

### 4. FastAPI Application Role

- Create role for Python environment setup
- Implement PostgreSQL installation and database creation
- Set up application deployment from Git
- Configure environment file with database connection
- Implement systemd service configuration

### 5. Playbook Development

- Create main playbook that includes all roles
- Implement proper role dependencies and ordering
- Add tags for selective execution
- Add pre-tasks for system preparation

### 6. Testing and Validation

- Create Vagrant configuration for Ansible testing
- Implement idempotence tests
- Validate all services are running correctly
- Test SSL certificate generation and configuration
- Verify security hardening measures

## Timeline Estimate

- **Week 1**: Infrastructure setup, Nginx role development
- **Week 2**: Cache services role, FastAPI application role
- **Week 3**: Playbook development, integration testing
- **Week 4**: Validation, documentation, and knowledge transfer

## Migration Success Criteria

1. All services (Nginx, Memcached, Redis, PostgreSQL, FastAPI) are properly installed and configured
2. Multiple Nginx sites are accessible with SSL
3. Security hardening measures are properly implemented
4. FastAPI application is running and connected to PostgreSQL
5. All Ansible roles are idempotent
6. Documentation is complete and accurate