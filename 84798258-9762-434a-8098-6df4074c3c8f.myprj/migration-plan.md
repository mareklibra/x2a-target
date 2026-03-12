# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 3-4 weeks
- Testing and Validation: 2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 7-8 weeks

**Complexity Assessment:** Medium to High
- Multiple interconnected services
- Security configurations that need careful migration
- SSL certificate management
- Database integration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall)

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx, ssl_certificate, memcached, redisio) - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Defines the run list and cookbook versions - will be replaced by Ansible playbooks
- `solo.json`: Contains node configuration data including nginx sites and security settings - will be converted to Ansible variables
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `Vagrantfile`: Defines the development VM (Fedora 42) - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisions the VM with Chef - will be replaced with Ansible provisioning

### Target Details

- **Operating System**: Fedora 42 (primary), with support for Ubuntu 18.04+ and CentOS 7+ based on cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or direct package installation and configuration
- **ssl_certificate (~> 2.1)**: Replace with Ansible's `openssl_*` modules for certificate management
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached` or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or direct package installation

### Security Considerations

- **SSL Certificate Management**: Migration must maintain proper certificate generation and permissions
  - Current implementation uses self-signed certificates
  - Ensure proper file permissions (640) and ownership (root:ssl-cert) are maintained
  - Consider using Ansible's `openssl_certificate` module

- **Firewall Configuration (UFW)**: 
  - Current implementation enables UFW with default deny policy
  - Allows SSH, HTTP, and HTTPS
  - Use Ansible's `ufw` module to maintain identical configuration

- **Fail2ban Integration**:
  - Current implementation installs and configures fail2ban
  - Use Ansible's `template` module to create identical jail.local configuration

- **SSH Hardening**:
  - Current implementation disables root login and password authentication
  - Use Ansible's `lineinfile` or `template` module to configure SSH server

- **Redis Authentication**:
  - Current implementation sets a password for Redis
  - Ensure password is stored securely in Ansible Vault

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Challenge: Dynamically creating multiple virtual hosts with SSL
  - Solution: Use Ansible loops with templates to generate site configurations

- **SSL Certificate Generation**:
  - Challenge: Maintaining proper permissions and ownership
  - Solution: Use Ansible's `openssl_*` modules with appropriate file permissions

- **PostgreSQL User and Database Creation**:
  - Challenge: Securely creating database users and granting permissions
  - Solution: Use Ansible's `postgresql_*` modules with credentials in Ansible Vault

- **Service Orchestration**:
  - Challenge: Ensuring services start in the correct order with proper dependencies
  - Solution: Use Ansible handlers and meta dependencies to manage service ordering

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Start with basic Nginx installation and configuration
   - Add SSL certificate management
   - Add security hardening (fail2ban, ufw)
   - Add multi-site configuration

2. **cache** (Priority 2)
   - Memcached configuration
   - Redis installation and security configuration

3. **fastapi-tutorial** (Priority 3)
   - PostgreSQL installation and configuration
   - Python environment setup
   - Application deployment
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions.
2. Self-signed certificates are acceptable for the migrated solution (production would likely use Let's Encrypt or other CA).
3. The security requirements will remain the same (fail2ban, ufw, SSH hardening).
4. The FastAPI application repository will remain available at the specified URL.
5. The Redis password in the current configuration is for development purposes and will be replaced with a secure password stored in Ansible Vault.
6. The PostgreSQL credentials in the FastAPI configuration are for development and will be replaced with secure credentials in Ansible Vault.
7. The current Vagrant setup will be maintained for development and testing.