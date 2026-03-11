# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx configuration with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- Well-structured Chef cookbooks with clear separation of concerns
- Standard infrastructure components (Nginx, Redis, Memcached, PostgreSQL)
- Security configurations that need careful migration
- Hardcoded secrets that should be moved to Ansible Vault

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled subdomains, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening with fail2ban and UFW

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Chef policy file defining the run list - will be replaced by Ansible playbooks
- `Vagrantfile`: VM configuration for development/testing - can be adapted for Ansible testing
- `solo.json`: Chef node attributes - will be converted to Ansible variables
- `solo.rb`: Chef configuration - will be replaced by ansible.cfg
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible provisioning

### Target Details

Based on the source configuration files:

- **Operating System**: Supports Ubuntu 18.04+ and CentOS 7+, with Fedora 42 used in Vagrant development environment
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud VMs

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation
- **ssl_certificate (~> 2.1)**: Replace with Ansible OpenSSL modules for certificate generation

### Security Considerations

- **SSL Certificate Management**: Currently using self-signed certificates; migrate to Ansible's crypto modules
- **Fail2ban Configuration**: Migrate fail2ban configuration to Ansible
- **UFW Firewall Rules**: Convert UFW rules to Ansible ufw module
- **SSH Hardening**: Migrate SSH security configurations
- **Redis Authentication**: Move Redis password to Ansible Vault
- **PostgreSQL Credentials**: Move database credentials to Ansible Vault
- **Sysctl Security Settings**: Migrate sysctl security configurations

### Technical Challenges

- **Multi-site Nginx Configuration**: Ensure proper templating of Nginx site configurations
- **SSL Certificate Generation**: Maintain proper permissions and security for SSL certificates
- **Service Dependencies**: Ensure proper ordering of service deployments (e.g., PostgreSQL before FastAPI app)
- **Idempotent Execution**: Ensure database creation and user setup is idempotent

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Base Nginx installation
   - SSL certificate generation
   - Site configurations
   - Security hardening

2. **cache** (low complexity, independent service)
   - Memcached configuration
   - Redis installation and configuration

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - PostgreSQL installation and configuration
   - Python environment setup
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Vagrant-based for development/testing
2. Self-signed certificates are acceptable for the migrated solution
3. The same operating system support (Ubuntu 18.04+, CentOS 7+) is required
4. The FastAPI application source code will remain at the same GitHub repository
5. The current security configurations (fail2ban, UFW, SSH hardening) are required in the migrated solution
6. Redis and PostgreSQL passwords should be secured in the migrated solution
7. The same network configuration (ports, IPs) will be maintained

## Ansible Structure Recommendation

```
ansible-nginx-multisite/
├── ansible.cfg
├── inventory/
│   ├── development
│   └── production
├── group_vars/
│   ├── all/
│   │   ├── main.yml
│   │   └── vault.yml
│   └── webservers/
│       └── main.yml
├── host_vars/
├── roles/
│   ├── nginx-multisite/
│   ├── cache/
│   └── fastapi-tutorial/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
└── Vagrantfile
```

## Migration Steps

1. **Setup Ansible Project Structure**
   - Create directory structure
   - Configure ansible.cfg
   - Create inventory files

2. **Create Ansible Vault for Secrets**
   - Store Redis password
   - Store PostgreSQL credentials
   - Store any other sensitive information

3. **Develop Ansible Roles**
   - Convert each Chef cookbook to an Ansible role
   - Create templates from Chef templates
   - Convert Chef attributes to Ansible variables

4. **Create Playbooks**
   - Create main site.yml playbook
   - Create individual service playbooks

5. **Test and Validate**
   - Test with Vagrant
   - Verify all functionality matches original Chef implementation

6. **Document**
   - Create README with usage instructions
   - Document variables and customization options