# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for managing a multi-site Nginx web server with caching capabilities (Memcached and Redis). The migration to Ansible is estimated to be of medium complexity, requiring approximately 2-3 weeks of effort for a single engineer or 1-2 weeks for a small team. The primary components are two Chef cookbooks: `nginx-multisite` and `cache`, with dependencies on external cookbooks from the Chef Supermarket.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and firewall configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: 
      - Multi-site configuration with SSL support
      - Self-signed certificate generation
      - Security hardening (fail2ban, UFW firewall)
      - System-level security configurations
      - Custom static content for each site

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features:
      - Memcached configuration
      - Redis with password authentication
      - Log directory management
      - Configuration file manipulation

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies from Chef Supermarket (nginx, ssl_certificate, memcached, redisio)
- `Policyfile.rb`: Defines the Chef policy with run list and cookbook dependencies
- `Policyfile.lock.json`: Locked versions of all dependencies
- `solo.json`: Configuration data for Chef Solo, including site configurations and security settings
- `solo.rb`: Chef Solo configuration
- `Vagrantfile`: Defines a development VM using Ubuntu 22.04 with port forwarding
- `vagrant-provision.sh`: Bash script to provision the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Ubuntu 22.04 LTS (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (specified in Vagrantfile)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **ssl_certificate (~> 2.1)**: Replace with Ansible's openssl_* modules for certificate management
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks
- **selinux (6.2.4)**: Replace with Ansible's selinux module for SELinux management

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migrate to Ansible's `openssl_certificate` module or consider integrating with Let's Encrypt using `community.crypto.acme_certificate`.
- **Firewall Configuration (UFW)**: Replace with Ansible's `community.general.ufw` module.
- **Fail2Ban Configuration**: Use Ansible to deploy and configure fail2ban with similar jail settings.
- **SSH Hardening**: Migrate SSH security configurations using Ansible's `lineinfile` or `template` modules.
- **System Hardening (sysctl)**: Use Ansible's `sysctl` module to apply the same kernel parameter security settings.
- **Redis Authentication**: Ensure Redis password is stored securely using Ansible Vault.

### Technical Challenges

- **Custom Resource Migration**: The `lineinfile` custom resource will need to be replaced with Ansible's native `lineinfile` module.
- **Template Conversion**: All ERB templates need to be converted to Jinja2 format for Ansible.
- **Multi-site Configuration**: The dynamic site configuration will need careful migration to ensure all sites are properly configured.
- **SSL Certificate Handling**: Ensuring proper certificate generation and permissions in Ansible.
- **Redis Configuration Manipulation**: The Ruby block that modifies Redis configuration will need to be reimplemented using Ansible's `lineinfile` or `replace` modules.

### Migration Order

1. **Base Infrastructure** (low risk, foundation)
   - Vagrant environment
   - Basic system configuration

2. **Nginx Base Configuration** (medium complexity)
   - Basic Nginx installation and configuration
   - Security configuration

3. **SSL Certificate Management** (medium complexity)
   - Certificate generation
   - Directory permissions

4. **Multi-site Configuration** (medium complexity)
   - Virtual host configuration
   - Site-specific content

5. **Security Hardening** (medium complexity)
   - Fail2ban
   - UFW firewall
   - System hardening

6. **Caching Services** (high complexity)
   - Memcached configuration
   - Redis installation and security

### Assumptions

1. The target environment will continue to be Ubuntu 22.04 LTS.
2. Self-signed certificates are acceptable for the migrated solution (no requirement for Let's Encrypt or commercial certificates).
3. The same security hardening measures are required in the Ansible implementation.
4. The Redis password in the current implementation is for development purposes and will be replaced with a secure password stored in Ansible Vault.
5. The current implementation is for a development/testing environment as indicated by the use of Vagrant and self-signed certificates.
6. The multi-site configuration will maintain the same structure with three virtual hosts (test, ci, status).
7. SELinux is used in the environment despite the primary OS being Ubuntu (which typically uses AppArmor).

## Ansible Structure Recommendation

```
ansible/
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── all.yml
│       └── webservers.yml
├── roles/
│   ├── nginx-multisite/
│   │   ├── defaults/main.yml
│   │   ├── handlers/main.yml
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── nginx.yml
│   │   │   ├── security.yml
│   │   │   ├── ssl.yml
│   │   │   └── sites.yml
│   │   ├── templates/
│   │   │   ├── nginx.conf.j2
│   │   │   ├── security.conf.j2
│   │   │   ├── site.conf.j2
│   │   │   ├── fail2ban.jail.local.j2
│   │   │   └── sysctl-security.conf.j2
│   │   └── files/
│   │       ├── ci/index.html
│   │       ├── status/index.html
│   │       └── test/index.html
│   └── cache/
│       ├── defaults/main.yml
│       ├── handlers/main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── memcached.yml
│       │   └── redis.yml
│       └── templates/
│           └── redis.conf.j2
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   └── cache.yml
└── vagrant/
    └── Vagrantfile
```

## Migration Timeline Estimate

- **Analysis and Planning**: 1-2 days
- **Infrastructure Setup**: 1 day
- **Nginx Base Configuration**: 2-3 days
- **SSL and Multi-site Configuration**: 2-3 days
- **Security Hardening**: 2-3 days
- **Caching Services**: 2-3 days
- **Testing and Validation**: 2-3 days
- **Documentation**: 1 day

**Total Estimated Time**: 11-18 days (2-3 weeks) for a single engineer