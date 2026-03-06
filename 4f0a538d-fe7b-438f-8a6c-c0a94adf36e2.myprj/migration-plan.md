# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for deploying a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary cookbooks, handling external dependencies, and ensuring security configurations are maintained. Based on the complexity and scope, this migration is estimated to require 3-4 weeks with a team of 2 engineers.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and fail2ban integration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security headers, rate limiting, UFW firewall configuration

- **cache**:
    - Description: Configures Redis and Memcached caching services with security settings
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx, ssl_certificate, memcached, redisio)
- `Policyfile.rb`: Defines the run list and cookbook dependencies
- `Vagrantfile`: Configures a Fedora 42 VM for development/testing
- `solo.rb`: Chef Solo configuration
- `solo.json`: Defines node attributes including Nginx site configurations and security settings
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration
- **ssl_certificate (~> 2.1)**: Replace with Ansible OpenSSL modules for certificate generation

### Security Considerations

- **SSL/TLS Configuration**: Maintain strong cipher configurations and protocols (TLSv1.2, TLSv1.3)
- **Fail2ban Integration**: Ensure fail2ban configurations are migrated with appropriate jails
- **UFW Firewall Rules**: Maintain firewall configurations with Ansible ufw module
- **Security Headers**: Preserve HTTP security headers in Nginx configurations
- **System Hardening**: Maintain sysctl security configurations
- **SSH Hardening**: Preserve SSH security configurations (disable root login, password authentication)
- **Redis Authentication**: Maintain Redis password authentication
- **PostgreSQL Security**: Ensure database user permissions are properly configured

### Technical Challenges

- **Template Conversion**: Chef ERB templates need to be converted to Jinja2 for Ansible
- **Resource Mapping**: Chef resources need to be mapped to appropriate Ansible modules
- **Idempotency**: Ensure all Ansible tasks maintain the idempotency of the original Chef recipes
- **Secrets Management**: Implement Ansible Vault for sensitive data (Redis password, PostgreSQL credentials)
- **Service Dependencies**: Maintain proper ordering of service installations and configurations

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Base Nginx installation and configuration
   - SSL certificate generation
   - Virtual host configuration
   - Security hardening (fail2ban, firewall, headers)

2. **cache** (low complexity, standalone services)
   - Memcached installation and configuration
   - Redis installation and configuration with authentication

3. **fastapi-tutorial** (high complexity, application deployment)
   - PostgreSQL installation and database setup
   - Python environment configuration
   - Application deployment from Git
   - Systemd service configuration

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distribution
2. The same security requirements will apply in the Ansible implementation
3. Self-signed certificates are acceptable for development (production would likely use Let's Encrypt)
4. The FastAPI application repository will remain available at the specified URL
5. The multi-site configuration with test.cluster.local, ci.cluster.local, and status.cluster.local will be maintained
6. Redis will continue to require password authentication
7. The PostgreSQL database structure required by the FastAPI application is created by the application itself

## Ansible Structure Recommendation

```
ansible-nginx-multisite/
├── inventories/
│   ├── development/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
├── roles/
│   ├── nginx-multisite/
│   │   ├── tasks/
│   │   ├── templates/
│   │   ├── files/
│   │   └── defaults/
│   ├── cache/
│   │   ├── tasks/
│   │   ├── templates/
│   │   └── defaults/
│   └── fastapi-tutorial/
│       ├── tasks/
│       ├── templates/
│       └── defaults/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
└── vagrant/
    └── Vagrantfile
```

## Testing Strategy

1. Develop and test each role independently using Molecule
2. Create integration tests to verify interactions between roles
3. Use the existing Vagrantfile as a basis for a test environment
4. Implement CI/CD pipeline for automated testing

## Timeline Estimate

- **Week 1**: Analysis and planning, role structure setup
- **Week 2**: Nginx and security configuration migration
- **Week 3**: Cache services and FastAPI application migration
- **Week 4**: Testing, documentation, and knowledge transfer

## Migration Team Recommendations

- 1 Senior DevOps Engineer with Ansible expertise
- 1 Application Developer familiar with FastAPI and PostgreSQL
- Periodic review from Security Engineer to validate security configurations