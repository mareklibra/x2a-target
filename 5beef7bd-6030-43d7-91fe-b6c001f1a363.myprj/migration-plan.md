# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backend with PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3-4 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7-8 weeks

**Complexity Assessment:** Medium
- Multiple interconnected services
- Security configurations that need careful migration
- Database and application deployment requirements

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site-specific configurations
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
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Chef policy definition - will be replaced by Ansible playbooks
- `solo.json`: Chef node configuration - will be converted to Ansible inventory variables
- `solo.rb`: Chef configuration - will be replaced by ansible.cfg
- `Vagrantfile`: Development environment definition - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisioning script - will be replaced by Ansible playbook calls

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in Vagrant development environment
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role (e.g., geerlingguy.nginx)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., geerlingguy.memcached)
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (e.g., geerlingguy.redis)
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management modules (openssl_*)

### Security Considerations

- **SSL Certificate Management**: Migration must maintain proper certificate generation and permissions
  - Use Ansible's `openssl_certificate`, `openssl_privatekey`, and `openssl_csr` modules
  - Ensure proper file permissions (640 for private keys, ssl-cert group ownership)

- **Firewall Configuration (UFW)**: Maintain security rules
  - Use Ansible's `ufw` module to configure firewall rules
  - Ensure SSH, HTTP, and HTTPS ports remain accessible

- **fail2ban Configuration**: Maintain brute force protection
  - Use Ansible's `template` module to configure fail2ban
  - Ensure proper service restart handlers

- **SSH Hardening**: Maintain secure SSH configuration
  - Use Ansible's `lineinfile` or dedicated SSH role to configure SSH daemon
  - Maintain settings for root login and password authentication

- **Redis Authentication**: Maintain password protection
  - Ensure Redis password is stored securely (Ansible Vault)
  - Configure Redis with authentication in Ansible

### Technical Challenges

- **Multi-site Nginx Configuration**: The current setup dynamically generates site configurations
  - Solution: Use Ansible templates with loops over site variables
  - Ensure proper SSL certificate paths and configurations

- **PostgreSQL User and Database Setup**: The current setup uses direct psql commands
  - Solution: Use Ansible's `postgresql_*` modules for idempotent database management
  - Ensure proper password handling with Ansible Vault

- **Python Application Deployment**: The current setup uses Git and venv
  - Solution: Use Ansible's `git` and `pip` modules
  - Ensure proper systemd service configuration

- **Redis Configuration Workarounds**: The current setup includes a hack to fix Redis configuration
  - Solution: Create proper Redis configuration template in Ansible
  - Test thoroughly to ensure compatibility

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component
   - Other services depend on it
   - Contains security configurations

2. **cache** (Priority 2)
   - Supporting services for the application
   - Moderate complexity with Redis authentication

3. **fastapi-tutorial** (Priority 3)
   - Application deployment
   - Depends on properly configured infrastructure

### Assumptions

1. The target environment will continue to be either Ubuntu (>= 18.04) or CentOS (>= 7.0)
2. The same security requirements will apply in the new environment
3. The FastAPI application repository will remain accessible at the same URL
4. The multi-site configuration will remain similar (test.cluster.local, ci.cluster.local, status.cluster.local)
5. Self-signed certificates are acceptable for development (production would likely use Let's Encrypt)
6. Redis and Memcached configurations will remain similar
7. PostgreSQL will be installed locally on the same server

## Ansible Structure Recommendation

```
ansible-nginx-multisite/
├── ansible.cfg
├── inventory/
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── webservers.yml
│   └── hosts
├── roles/
│   ├── nginx_multisite/
│   ├── cache_services/
│   └── fastapi_app/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
├── templates/
├── files/
└── requirements.yml
```

## Next Steps

1. Create a detailed inventory structure with variables extracted from solo.json
2. Develop the nginx_multisite role first, with templates for site configurations
3. Create the cache_services role for Redis and Memcached
4. Develop the fastapi_app role for application deployment
5. Test each role individually in a Vagrant environment
6. Create a comprehensive playbook that applies all roles
7. Document the new Ansible structure and usage