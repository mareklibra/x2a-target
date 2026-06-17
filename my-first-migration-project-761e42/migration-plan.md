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
- The repository has well-structured Chef cookbooks with clear dependencies
- Security configurations are present and need careful migration
- External cookbook dependencies need to be replaced with Ansible Galaxy roles or custom implementations
- Secrets management needs to be addressed (Redis password, PostgreSQL credentials)

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall), sysctl security settings

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, service management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx, memcached, redisio) - will be replaced by Ansible Galaxy requirements.yml
- `solo.json`: Contains node configuration and run list - will be replaced by Ansible inventory variables
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `Vagrantfile`: Defines development VM using Fedora 42 - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisions the VM with Chef - will be replaced with Ansible provisioning

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (based on cookbook metadata)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` module or community.general collection
- **memcached (~> 6.0)**: Replace with Ansible's `memcached` module or dedicated role
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (possibly from Ansible Galaxy)
- **PostgreSQL**: Replace with Ansible's `postgresql` module or dedicated role

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW - migrate to Ansible's `ufw` module
- **fail2ban**: Migrate fail2ban configuration to Ansible using the `fail2ban` module
- **SSH Hardening**: Migrate SSH security settings (disable root login, password authentication) using Ansible's `lineinfile` or templates
- **sysctl Security Settings**: Migrate using Ansible's `sysctl` module
- **Vault/secrets management**:
  - Redis authentication password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL credentials in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - Consider using Ansible Vault for these credentials

### Technical Challenges

- **SSL Certificate Generation**: The Chef cookbook generates self-signed certificates - implement equivalent functionality using Ansible's `openssl_certificate` module
- **Multi-site Configuration**: The dynamic generation of Nginx site configurations based on node attributes needs careful translation to Ansible templates and variables
- **Service Dependencies**: Ensure proper ordering of service installations and configurations (e.g., PostgreSQL before FastAPI application)
- **Idempotency**: Ensure all Ansible tasks are idempotent, particularly the database user and database creation tasks

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Add security hardening (fail2ban, ufw, sysctl)
   - Implement multi-site configuration

2. **cache** (Priority 2)
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Ensure proper service management

3. **fastapi-tutorial** (Priority 3)
   - Implement PostgreSQL installation and configuration
   - Set up Python environment and dependencies
   - Deploy application from Git
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions (Ubuntu 18.04+, CentOS 7+)
2. Self-signed certificates are acceptable for development/testing (production would likely use Let's Encrypt or other CA)
3. The current security configurations (fail2ban, ufw, SSH hardening) are appropriate for the target environment
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. The current hardcoded credentials will be replaced with more secure credential management in Ansible
6. The Nginx sites configuration in solo.json (test.cluster.local, ci.cluster.local, status.cluster.local) represents the desired state
7. The current VM resources (2GB RAM, 2 CPUs) are sufficient for the application stack