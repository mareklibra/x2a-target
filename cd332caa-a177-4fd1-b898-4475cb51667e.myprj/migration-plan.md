# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary cookbooks, handling security configurations, and ensuring proper service orchestration. Based on the complexity and scope, this migration is estimated to require 3-4 weeks with 1-2 engineers.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and firewall configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site SSL configuration, fail2ban integration, UFW firewall rules, security headers

- **cache**:
    - Description: Configures Redis and Memcached caching services with security settings
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx, ssl_certificate, memcached, redisio) - will be replaced by Ansible Galaxy requirements
- `Policyfile.rb`: Defines the run list and cookbook versions - will be replaced by Ansible playbook structure
- `Vagrantfile`: Defines the development VM configuration - can be adapted for Ansible testing
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `solo.json`: Contains node attributes and run list - will be converted to Ansible variables
- `vagrant-provision.sh`: Provisions the Vagrant VM with Chef - will be replaced with Ansible provisioning

### Target Details

Based on the source repository analysis:

- **Operating System**: Fedora 42 (from Vagrantfile) with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Libvirt (from Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **ssl_certificate (~> 2.1)**: Replace with Ansible crypto modules for certificate generation
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration

### Security Considerations

- **SSL/TLS Configuration**: Migrate the SSL certificate generation and configuration, ensuring proper permissions and security settings
- **Firewall Rules**: Convert UFW rules to appropriate Ansible firewall module (ufw or firewalld depending on target OS)
- **fail2ban Configuration**: Migrate fail2ban jail configurations using Ansible templates
- **System Hardening**: Convert sysctl security settings using Ansible sysctl module
- **SSH Hardening**: Migrate SSH security configurations using Ansible's openssh_config module
- **Redis Authentication**: Ensure Redis password is properly managed through Ansible Vault

### Technical Challenges

- **Multi-site Configuration**: The Nginx configuration manages multiple virtual hosts with SSL - ensure proper template conversion
- **Custom Resource**: The `lineinfile` custom resource needs to be replaced with Ansible's lineinfile module
- **Service Dependencies**: Ensure proper ordering of service deployments (PostgreSQL before FastAPI, etc.)
- **Secret Management**: Redis password and PostgreSQL credentials need secure handling in Ansible Vault

### Migration Order

1. Base infrastructure (system packages, security configurations)
2. Cache services (Redis and Memcached)
3. Nginx configuration with SSL
4. FastAPI application with PostgreSQL

### Assumptions

- The target environment will continue to use the same operating systems (Fedora/Ubuntu/CentOS)
- The application architecture will remain the same (Nginx + FastAPI + PostgreSQL + Redis/Memcached)
- Self-signed certificates are acceptable for development (production would likely use Let's Encrypt or other CA)
- The current security configurations are appropriate and should be maintained

## Detailed Migration Tasks

### 1. Project Structure Setup

```
ansible-nginx-multisite/
├── inventories/
│   ├── development/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   └── production/
├── roles/
│   ├── nginx-multisite/
│   ├── cache/
│   └── fastapi-tutorial/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
├── requirements.yml
└── Vagrantfile
```

### 2. Role Conversion Tasks

#### nginx-multisite Role

- Convert Nginx configuration templates to Ansible templates
- Migrate SSL certificate generation to Ansible crypto modules
- Convert security configurations (fail2ban, UFW, sysctl) to appropriate Ansible modules
- Migrate virtual host configurations to Ansible templates

#### cache Role

- Replace Chef memcached cookbook with Ansible memcached configuration
- Replace Chef redisio cookbook with Ansible Redis configuration
- Secure Redis password in Ansible Vault

#### fastapi-tutorial Role

- Convert Python environment setup to Ansible pip and venv modules
- Migrate PostgreSQL setup to Ansible PostgreSQL modules
- Convert systemd service configuration to Ansible systemd module

### 3. Variable Management

- Convert Chef attributes to Ansible variables
- Move sensitive data to Ansible Vault
- Create appropriate group_vars and host_vars structure

### 4. Testing Strategy

- Update Vagrantfile to use Ansible provisioner
- Create test playbooks for each role
- Implement molecule testing for roles

## Timeline Estimate

- **Week 1**: Project setup, role structure, and base configurations
- **Week 2**: Nginx and security configurations
- **Week 3**: Cache services and FastAPI application
- **Week 4**: Testing, documentation, and handover

## Migration Risks

- Security configurations may need adjustment for different target environments
- Custom Chef resources may not have direct Ansible equivalents
- Service interdependencies may require careful orchestration
- SSL certificate management differs between Chef and Ansible approaches