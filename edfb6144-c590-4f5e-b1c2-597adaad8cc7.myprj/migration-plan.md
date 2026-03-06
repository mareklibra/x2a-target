# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will require converting 3 Chef cookbooks with their recipes, templates, and resources to equivalent Ansible roles and playbooks. The estimated complexity is medium, with security configurations and SSL certificate management requiring special attention. The estimated timeline for migration is 2-3 weeks for a single developer or 1 week for a team of 2-3.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and fail2ban integration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening, UFW firewall configuration

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
- `Policyfile.rb`: Defines the Chef policy with run list and cookbook dependencies
- `Vagrantfile`: Configures a Fedora 42 VM for local development and testing
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `solo.rb`: Chef Solo configuration file
- `solo.json`: Chef Solo JSON attributes file with site configurations and security settings

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role or package installation tasks
- **ssl_certificate (~> 2.1)**: Replace with Ansible OpenSSL modules for certificate generation

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migrate to Ansible's `openssl_*` modules or consider integrating with Let's Encrypt via `community.crypto.acme_certificate`.
- **Fail2ban Configuration**: Convert fail2ban jail templates to Ansible templates with equivalent configuration.
- **UFW Firewall Rules**: Replace UFW command execution with Ansible's `community.general.ufw` module.
- **SSH Hardening**: Convert SSH security configurations to use Ansible's `lineinfile` or template modules.
- **Sysctl Security Settings**: Migrate sysctl security configurations to use Ansible's `posix.sysctl` module.
- **Redis Password**: The Redis password is hardcoded in the recipe. Use Ansible Vault to secure this credential.

### Technical Challenges

- **Custom Resource Migration**: The `lineinfile` custom resource will need to be replaced with Ansible's built-in `lineinfile` module.
- **Multi-site Configuration**: The dynamic generation of multiple Nginx sites will need careful translation to Ansible's templating system.
- **SSL Certificate Generation**: The self-signed certificate generation logic needs to be migrated to Ansible's OpenSSL modules.
- **PostgreSQL User/Database Creation**: The current implementation uses direct `psql` commands. This should be replaced with Ansible's `community.postgresql` modules.

### Migration Order

1. **cache cookbook** (Low complexity): Simple Redis and Memcached configuration
2. **nginx-multisite cookbook** (Medium complexity): Core Nginx configuration with security settings
3. **fastapi-tutorial cookbook** (Medium complexity): Application deployment with database dependencies

### Assumptions

1. The target environment will continue to be Fedora 42 or a compatible Linux distribution.
2. The self-signed SSL certificates are acceptable for the target environment, or a proper CA will be integrated later.
3. The current security configurations are appropriate for the target environment.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
5. The PostgreSQL database configuration (users, passwords) can remain the same in the migrated solution.

## Detailed Migration Tasks

### 1. Project Structure Setup

```
ansible-nginx-multisite/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── roles/
│   ├── nginx-multisite/
│   ├── cache/
│   └── fastapi-tutorial/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
└── README.md
```

### 2. Role: nginx-multisite

Convert the nginx-multisite cookbook to an Ansible role with:

- Tasks for installing Nginx
- Templates for site configurations
- SSL certificate generation
- Security configurations (fail2ban, UFW, sysctl)
- Site content deployment

### 3. Role: cache

Convert the cache cookbook to an Ansible role with:

- Memcached installation and configuration
- Redis installation and configuration with secure password

### 4. Role: fastapi-tutorial

Convert the fastapi-tutorial cookbook to an Ansible role with:

- Python and system dependencies installation
- Git repository cloning
- Virtual environment setup
- PostgreSQL database configuration
- Environment file creation
- Systemd service configuration

### 5. Variable Management

Create variable files to replace Chef attributes:

- Extract site configurations to group_vars
- Move security settings to appropriate variable files
- Use Ansible Vault for sensitive information (Redis password, PostgreSQL credentials)

### 6. Testing Strategy

1. Create a Vagrant configuration for Ansible to match the current Chef setup
2. Test each role individually
3. Test the complete playbook
4. Verify all sites are accessible and properly configured

## Timeline Estimate

- **Analysis and Planning**: 1-2 days
- **Role Development**:
  - nginx-multisite: 2-3 days
  - cache: 1 day
  - fastapi-tutorial: 1-2 days
- **Integration and Testing**: 2-3 days
- **Documentation and Handover**: 1 day

**Total Estimated Time**: 8-12 working days for a single developer