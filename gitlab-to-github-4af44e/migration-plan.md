# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure for deploying a multi-site Nginx web server with SSL, caching services (Redis and Memcached), and a FastAPI Python application with PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks, handling external dependencies, and ensuring proper security configurations are maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 3-4 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 6-8 weeks

**Complexity Assessment:** Medium
- Multiple interconnected services
- Security configurations that need careful migration
- Database and application deployment with specific configurations

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening with fail2ban and UFW

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Defines Chef policy with run list - will be replaced by Ansible playbooks
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `solo.json`: Node attributes and run list - will be replaced by Ansible inventory and variables
- `Vagrantfile`: Defines development VM - can be adapted for Ansible testing
- `vagrant-provision.sh`: VM provisioning script - will be replaced by Ansible provisioning

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in Vagrant development environment
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **ssl_certificate (~> 2.1)**: Replace with Ansible crypto modules for certificate management
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package configuration

### Security Considerations

- **SSL/TLS Configuration**: 
  - Self-signed certificates are generated for development
  - Migration should maintain the same certificate structure and permissions
  - Consider integrating with Ansible crypto modules for certificate management

- **Firewall Configuration**: 
  - UFW is configured to allow only SSH, HTTP, and HTTPS
  - Migration should use Ansible's ufw or firewalld modules based on target OS

- **SSH Hardening**:
  - Root login is disabled
  - Password authentication is disabled
  - Migration should maintain these security practices

- **System Hardening**:
  - fail2ban is configured for brute force protection
  - sysctl security settings are applied
  - Migration should maintain these security configurations

- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook
  - Migration should use Ansible Vault for credential storage

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - The current implementation uses Chef templates and attributes to configure multiple sites
  - Challenge: Recreating the dynamic site configuration in Ansible
  - Solution: Use Ansible templates with loops over site configurations in variables

- **SSL Certificate Generation**:
  - Self-signed certificates are generated with specific ownership and permissions
  - Challenge: Ensuring proper certificate generation and permissions
  - Solution: Use Ansible's openssl_* modules with appropriate file permissions

- **Service Dependencies**:
  - The FastAPI application depends on PostgreSQL
  - Challenge: Ensuring proper service ordering and dependencies
  - Solution: Use Ansible handlers and meta dependencies between roles

- **Redis Configuration Patching**:
  - The current implementation uses a ruby_block to patch Redis configuration
  - Challenge: Replicating this behavior in Ansible
  - Solution: Use Ansible's lineinfile or template module with proper validation

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation and configuration
   - Add SSL certificate generation
   - Add security hardening (fail2ban, UFW)
   - Add multi-site configuration

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, depends on database)
   - Implement PostgreSQL installation and configuration
   - Implement Python environment setup
   - Implement application deployment
   - Configure systemd service

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS/Fedora
2. Self-signed certificates are acceptable for development (production would likely use Let's Encrypt or other CA)
3. The security requirements will remain the same (SSH hardening, firewall, fail2ban)
4. The FastAPI application repository will remain available at the specified URL
5. The current directory structure in /opt and /var/www will be maintained
6. The Redis and PostgreSQL passwords are development passwords and will be replaced in production
7. The Vagrant development environment will continue to be used for testing