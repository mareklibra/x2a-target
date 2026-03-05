# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for deploying a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting Chef cookbooks, recipes, templates, and attributes to equivalent Ansible roles, playbooks, templates, and variables.

**Estimated Timeline:** 3-4 weeks
**Complexity:** Medium
**Team Size Recommendation:** 2-3 DevOps engineers

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and fail2ban integration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening, fail2ban integration

- **cache**:
    - Description: Configures Redis and Memcached caching services with security settings
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (both local and from Chef Supermarket)
- `Policyfile.rb`: Defines the run list and cookbook dependencies
- `Policyfile.lock.json`: Locked versions of cookbook dependencies
- `Vagrantfile`: Defines the development VM configuration (Fedora 42)
- `solo.rb`: Chef Solo configuration
- `solo.json`: Node attributes and run list for Chef Solo
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified (appears to be designed for local development with Vagrant)

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role `geerlingguy.nginx` or create a custom Nginx role
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or DavidWittman.redis
- **ssl_certificate (~> 2.1)**: Replace with Ansible's `openssl_*` modules for certificate management

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migrate to Ansible's `openssl_certificate` module or consider integrating with Let's Encrypt using `geerlingguy.certbot`.
- **Fail2ban Configuration**: Migrate fail2ban configuration to Ansible using either a dedicated role or tasks within the Nginx role.
- **Firewall Rules (UFW)**: Replace UFW configuration with Ansible's `ufw` module or consider using `firewalld` for Fedora.
- **System Hardening**: Migrate sysctl security settings using Ansible's `sysctl` module.
- **SSH Hardening**: Migrate SSH security configurations using Ansible's `lineinfile` or template modules.
- **Redis Password**: Ensure Redis password is stored securely using Ansible Vault.

### Technical Challenges

- **Template Conversion**: Chef ERB templates need to be converted to Jinja2 format for Ansible.
- **Attribute to Variable Mapping**: Chef node attributes need to be mapped to Ansible variables with appropriate defaults.
- **Resource Idempotence**: Ensure Ansible tasks maintain the idempotence of the original Chef resources.
- **Service Management**: Convert Chef service resources to Ansible service modules.
- **PostgreSQL Configuration**: Migrate PostgreSQL database creation and user management to Ansible's `postgresql_*` modules.

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Start with basic Nginx configuration and virtual hosts
   - Add SSL and security configurations

2. **cache** (Priority 2)
   - Relatively simple module with external dependencies
   - Implement Redis and Memcached configurations

3. **fastapi-tutorial** (Priority 3)
   - Application deployment with database dependencies
   - Requires Python environment setup and PostgreSQL configuration

### Assumptions

1. The target environment will remain Fedora-based (the current Vagrantfile specifies Fedora 42).
2. Self-signed certificates are acceptable for development (production would likely use Let's Encrypt or other CA).
3. The security requirements will remain the same (fail2ban, UFW, SSH hardening).
4. The FastAPI application repository will remain available at the specified URL.
5. The Redis password in the current configuration is for development only and will be replaced with a secure password stored in Ansible Vault.

## Ansible Structure Recommendation

```
ansible-nginx-multisite/
├── inventories/
│   ├── development/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
├── roles/
│   ├── nginx-multisite/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── tasks/
│   │   └── templates/
│   ├── cache/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── tasks/
│   │   └── templates/
│   └── fastapi-tutorial/
│       ├── defaults/
│       ├── handlers/
│       ├── tasks/
│       └── templates/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
├── requirements.yml
└── Vagrantfile
```

## Testing Strategy

1. **Unit Testing**: Use `ansible-lint` to validate syntax and best practices.
2. **Integration Testing**: Use Molecule for role testing.
3. **End-to-End Testing**: Use Vagrant to provision a test VM with the complete Ansible configuration.
4. **Comparison Testing**: Validate that the Ansible-provisioned environment matches the Chef-provisioned environment.

## Knowledge Transfer Plan

1. Document the mapping between Chef resources and Ansible modules.
2. Create a README for each Ansible role explaining its purpose and configuration options.
3. Provide examples of common configuration changes.
4. Document the variable structure and defaults.
5. Create a troubleshooting guide for common issues.