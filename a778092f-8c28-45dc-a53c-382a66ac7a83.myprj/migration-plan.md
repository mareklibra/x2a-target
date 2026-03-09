# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Memcached and Redis) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies on community cookbooks need to be replaced with Ansible Galaxy roles
- Security configurations require careful migration
- Multiple services with interdependencies increase complexity

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw)

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Chef policy definition - will be replaced by Ansible playbook structure
- `solo.json`: Chef node configuration - will be migrated to Ansible inventory variables
- `solo.rb`: Chef configuration - not needed in Ansible
- `Vagrantfile`: Development environment definition - can be adapted for Ansible provisioning
- `vagrant-provision.sh`: Provisioning script - will be replaced by Ansible provisioning in Vagrantfile

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or local development environments

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role such as `geerlingguy.nginx` or custom Ansible role
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role such as `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role such as `geerlingguy.redis` or `DavidWittman.redis`
- **ssl_certificate (~> 2.1)**: Replace with Ansible's `openssl_*` modules for certificate management

### Security Considerations

- **Firewall (UFW)**: Migrate to Ansible's `ufw` module for firewall management
- **Fail2ban**: Use Ansible to install and configure fail2ban with appropriate jails
- **SSH Hardening**: Migrate SSH security configurations using Ansible's `lineinfile` or templates
- **SSL Certificates**: Ensure secure handling of SSL certificates and private keys using Ansible Vault for sensitive data
- **Redis Authentication**: Store Redis password in Ansible Vault and use in templates
- **PostgreSQL Credentials**: Store database credentials in Ansible Vault

### Technical Challenges

- **Multi-site Nginx Configuration**: Create flexible Ansible templates for Nginx site configurations
- **Self-signed Certificate Generation**: Implement using Ansible's `openssl_*` modules
- **Service Interdependencies**: Ensure proper ordering of service installation and configuration
- **Idempotent Security Configurations**: Convert Chef's conditional execution to Ansible's idempotent approach
- **PostgreSQL User/Database Creation**: Ensure idempotent database operations using Ansible's PostgreSQL modules

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Base Nginx installation
   - Security configurations (fail2ban, ufw)
   - SSL certificate generation
   - Virtual host configuration

2. **cache** (low complexity, independent service)
   - Memcached installation and configuration
   - Redis installation and configuration with authentication

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - PostgreSQL installation and configuration
   - Python environment setup
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as specified in the Vagrantfile)
2. Self-signed certificates are acceptable for development/testing environments
3. The security requirements (fail2ban, ufw, SSH hardening) will remain the same
4. The FastAPI application repository will remain available at the specified URL
5. Redis and Memcached configurations don't require advanced clustering features
6. The current directory structure in the target environment (`/var/www/` for websites, `/opt/` for applications) will be maintained
7. The PostgreSQL database schema is managed by the FastAPI application and not by the infrastructure code
8. The current security credentials in the code are for development only and will be replaced with proper secrets management in production

## Ansible Structure Recommendation

```
ansible-nginx-multisite/
├── inventories/
│   ├── development/
│   │   ├── group_vars/
│   │   │   └── all.yml  # Development environment variables
│   │   └── hosts        # Development inventory
│   └── production/
│       ├── group_vars/
│       │   └── all.yml  # Production environment variables
│       └── hosts        # Production inventory
├── roles/
│   ├── nginx_multisite/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── tasks/
│   │   └── templates/
│   ├── cache_services/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── tasks/
│   │   └── templates/
│   └── fastapi_app/
│       ├── defaults/
│       ├── handlers/
│       ├── tasks/
│       └── templates/
├── playbooks/
│   ├── site.yml         # Main playbook
│   ├── nginx.yml        # Nginx-specific playbook
│   ├── cache.yml        # Cache services playbook
│   └── fastapi.yml      # FastAPI application playbook
├── requirements.yml     # Ansible Galaxy requirements
└── Vagrantfile          # For local development
```

## Testing Strategy

1. Develop and test each role independently using Molecule
2. Create integration tests to verify interactions between components
3. Use Vagrant for local testing of the complete environment
4. Implement CI/CD pipeline for automated testing
5. Create validation playbooks to verify the correct operation of each service after migration