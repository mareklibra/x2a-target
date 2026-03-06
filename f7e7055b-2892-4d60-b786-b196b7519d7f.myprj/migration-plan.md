# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI application with PostgreSQL. The migration to Ansible will involve converting Chef cookbooks, recipes, templates, and attributes to Ansible roles, playbooks, templates, and variables.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity:** Medium to High
- Multiple interconnected services
- Security configurations
- SSL certificate management
- Database integration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw), custom site templates

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, log directory management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx, ssl_certificate, memcached, redisio)
- `Policyfile.rb`: Defines Chef policy with run list and cookbook dependencies
- `Vagrantfile`: Defines development VM configuration using Fedora 42 with libvirt provider
- `solo.rb`: Chef Solo configuration file
- `solo.json`: Chef Solo attributes file with site configurations and security settings
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: libvirt (based on Vagrant provider configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or local development

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or collection
- **memcached (~> 6.0)**: Replace with Ansible memcached role or collection
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or collection
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management modules (openssl_*)

### Security Considerations

- **Firewall (ufw)**: Migrate to Ansible's `ufw` module for firewall management
- **fail2ban**: Migrate to Ansible's `template` module for fail2ban configuration
- **SSL/TLS Configuration**: Use Ansible's `openssl_*` modules for certificate generation and management
- **System Hardening**: Migrate sysctl security settings using Ansible's `sysctl` module
- **SSH Hardening**: Migrate SSH security configurations using Ansible's `lineinfile` or `template` modules
- **Redis Authentication**: Ensure secure password management using Ansible Vault for Redis password

### Technical Challenges

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Consider using Ansible's `openssl_*` modules or integrating with Let's Encrypt via certbot role.
- **Multi-site Configuration**: Ensure the Ansible implementation maintains the flexibility of the current multi-site setup.
- **Database User/Password Management**: Implement secure handling of database credentials using Ansible Vault.
- **Service Dependencies**: Maintain proper ordering of service installation and configuration, especially for the FastAPI application which depends on PostgreSQL.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Base Nginx installation and configuration
   - Security hardening (fail2ban, ufw)
   - SSL certificate generation
   - Virtual host configuration

2. **cache** (low complexity, independent service)
   - Memcached configuration
   - Redis installation and security configuration

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - PostgreSQL installation and configuration
   - Python environment setup
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as indicated in the Vagrantfile).
2. Self-signed certificates are acceptable for the migrated solution (production environments might require integration with Let's Encrypt or other certificate authorities).
3. The security requirements (fail2ban, ufw, SSH hardening) will remain the same in the Ansible implementation.
4. The FastAPI application source will continue to be pulled from the same Git repository.
5. The multi-site configuration with three virtual hosts (test.cluster.local, ci.cluster.local, status.cluster.local) will be maintained.
6. Redis will continue to require password authentication.
7. The current directory structure for web content (/var/www/[site]) will be maintained.
8. The PostgreSQL database configuration (user: fastapi, password: fastapi_password, database: fastapi_db) will remain the same.

## Ansible Structure Recommendation

```
ansible/
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml
│       └── web_servers.yml
├── roles/
│   ├── nginx_multisite/
│   ├── cache/
│   └── fastapi_app/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
└── templates/
    ├── nginx/
    ├── redis/
    └── fastapi/
```

## Security Recommendations

1. Use Ansible Vault for sensitive information (passwords, keys)
2. Consider implementing a more robust SSL certificate management solution
3. Implement proper user management and permissions
4. Consider adding monitoring and logging solutions
5. Implement regular security updates via Ansible