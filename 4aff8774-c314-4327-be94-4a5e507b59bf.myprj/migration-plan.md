# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for managing a multi-site Nginx web server with SSL configuration and caching services (Memcached and Redis). The migration to Ansible is estimated to be of medium complexity, requiring approximately 2-3 weeks of effort for a single engineer. The repository is well-structured with clear separation of concerns, making it suitable for a phased migration approach.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and firewall configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, UFW), system-level security settings

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

### Infrastructure Files

- `Berksfile`: Dependency management file listing cookbook dependencies (nginx, ssl_certificate, memcached, redisio)
- `Policyfile.rb`: Chef policy file defining the run list and cookbook dependencies
- `Policyfile.lock.json`: Locked versions of cookbook dependencies
- `solo.json`: Configuration data for Chef Solo, including site configurations and security settings
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Defines a development environment using Ubuntu 22.04 with port forwarding for HTTP/HTTPS
- `vagrant-provision.sh`: Provisioning script for the Vagrant environment

### Target Details

Based on the source configuration files:

- **Operating System**: Ubuntu 22.04 LTS (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (specified in Vagrantfile)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (12.3.1)**: Replace with Ansible nginx role or nginx_core module
- **ssl_certificate (2.1.0)**: Replace with Ansible openssl_* modules for certificate management
- **memcached (6.1.0)**: Replace with Ansible memcached role or package module with custom configuration
- **redisio (7.2.4)**: Replace with Ansible redis role or package module with custom configuration
- **selinux (6.2.4)**: Replace with Ansible selinux module for SELinux configuration

### Security Considerations

- **SSL Certificate Management**: Migration must preserve the self-signed certificate generation for development environments
- **Firewall Configuration**: UFW rules need to be migrated to equivalent Ansible ufw module tasks
- **fail2ban Configuration**: Custom fail2ban jails need to be preserved in the migration
- **SSH Hardening**: SSH configuration hardening (disable root login, password authentication) must be maintained
- **System Hardening**: Sysctl security settings must be preserved in the migration
- **Redis Authentication**: Redis password authentication must be maintained

### Technical Challenges

- **Multi-site Configuration**: The dynamic generation of Nginx site configurations based on node attributes needs careful translation to Ansible variables and templates
- **SSL Certificate Generation**: Self-signed certificate generation logic needs to be preserved
- **Security Hardening**: Comprehensive security configurations across multiple services need careful migration
- **Redis Configuration Workarounds**: The current implementation includes a Ruby block to fix Redis configuration, which will need a different approach in Ansible

### Migration Order

1. **Base Infrastructure** (low risk, foundation): Vagrant environment, basic system configuration
2. **nginx-multisite Cookbook** (moderate complexity): Core web server functionality
   a. Basic Nginx installation and configuration
   b. SSL certificate management
   c. Virtual host configuration
   d. Security hardening (fail2ban, firewall)
3. **cache Cookbook** (moderate complexity): Caching services
   a. Memcached configuration
   b. Redis installation and configuration

### Assumptions

1. The target environment will continue to be Ubuntu 22.04 LTS
2. Self-signed certificates are acceptable for development environments
3. The same security hardening measures will be required in the Ansible implementation
4. The multi-site configuration pattern will be preserved
5. Redis and Memcached will continue to be the caching solutions
6. The migration will not involve architectural changes to the application stack
7. No external service dependencies (databases, APIs) are present in the current implementation

## Detailed Implementation Plan

### 1. Ansible Directory Structure

Create the following Ansible directory structure:

```
ansible/
├── inventory/
│   ├── hosts
│   └── group_vars/
│       └── all.yml
├── roles/
│   ├── nginx-multisite/
│   │   ├── tasks/
│   │   ├── templates/
│   │   ├── files/
│   │   ├── handlers/
│   │   └── defaults/
│   └── cache/
│       ├── tasks/
│       ├── templates/
│       ├── handlers/
│       └── defaults/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   └── cache.yml
└── vagrant/
    └── Vagrantfile
```

### 2. Variable Migration

Convert Chef node attributes to Ansible variables in `group_vars/all.yml`:

```yaml
# Nginx sites configuration
nginx_sites:
  test.cluster.local:
    document_root: /var/www/test.cluster.local
    ssl_enabled: true
  ci.cluster.local:
    document_root: /var/www/ci.cluster.local
    ssl_enabled: true
  status.cluster.local:
    document_root: /var/www/status.cluster.local
    ssl_enabled: true

# SSL configuration
nginx_ssl:
  certificate_path: /etc/ssl/certs
  private_key_path: /etc/ssl/private

# Security configuration
security:
  fail2ban:
    enabled: true
  ufw:
    enabled: true
  ssh:
    disable_root: true
    password_auth: false

# Redis configuration
redis_servers:
  - port: 6379
    requirepass: redis_secure_password_123
```

### 3. Role Development

#### nginx-multisite Role

1. Convert Chef recipes to Ansible tasks
2. Convert ERB templates to Jinja2 templates
3. Implement handlers for service restarts
4. Implement SSL certificate generation using Ansible's openssl_* modules

#### cache Role

1. Implement Memcached configuration using Ansible package and template modules
2. Implement Redis configuration using Ansible package and template modules
3. Address the Redis configuration workaround using Ansible's lineinfile or template module

### 4. Testing Strategy

1. Use Vagrant for local testing of the Ansible playbooks
2. Implement molecule tests for individual roles
3. Create a CI pipeline for automated testing

### 5. Documentation

1. Create README files for each role explaining usage and variables
2. Document the migration process and any changes in behavior
3. Update the Vagrantfile to use Ansible provisioner instead of Chef

## Timeline Estimate

- **Week 1**: Setup Ansible structure, migrate variables, develop nginx-multisite role
- **Week 2**: Develop cache role, implement testing, refine configurations
- **Week 3**: Documentation, final testing, and handover

## Conclusion

The migration from Chef to Ansible for this infrastructure is straightforward but requires careful attention to security configurations and multi-site setup. The well-structured Chef codebase provides a clear path for migration, and the resulting Ansible code should maintain the same functionality while leveraging Ansible's idempotent approach to configuration management.