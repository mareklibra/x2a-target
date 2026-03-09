# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three primary Chef cookbooks to Ansible roles and playbooks, addressing external dependencies, and ensuring security configurations are properly maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- **Total**: 5-7 weeks

**Complexity Assessment**: Medium
- The Chef cookbooks are well-structured and follow standard patterns
- Security configurations need careful attention during migration
- Multiple services with interdependencies require coordinated testing

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
    - Key Features: Python virtual environment setup, PostgreSQL database provisioning, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements.yml
- `Policyfile.rb`: Chef policy definition - will be replaced by Ansible playbooks
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `solo.json`: Node attributes and run list - will be replaced by Ansible inventory and variables
- `Vagrantfile`: Development environment definition - can be adapted for Ansible testing
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible provisioning

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) based on cookbook metadata. The Vagrantfile uses Fedora 42 as the development environment.
- **Virtual Machine Technology**: Vagrant with libvirt provider for development/testing
- **Cloud Platform**: Not specified in the repository, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role (e.g., geerlingguy.nginx)
- **memcached (~> 6.0)**: Replace with Ansible memcached role (e.g., geerlingguy.memcached)
- **redisio (~> 7.2.4)**: Replace with Ansible Redis role (e.g., geerlingguy.redis)
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management modules (openssl_* modules)

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migration should maintain this capability while allowing for integration with Let's Encrypt or other certificate authorities.
- **Firewall Configuration (UFW)**: Current implementation configures UFW with specific rules. Ansible has dedicated UFW modules to maintain this functionality.
- **fail2ban Configuration**: Current implementation installs and configures fail2ban. Ansible has modules to manage fail2ban configuration.
- **SSH Hardening**: Current implementation disables root login and password authentication. Ansible has modules to manage SSH configuration.
- **Redis Authentication**: Current implementation sets a Redis password. This should be migrated to use Ansible Vault for secure storage.

### Technical Challenges

- **Multi-site Nginx Configuration**: The current implementation dynamically generates site configurations based on node attributes. Ansible templates will need to replicate this dynamic behavior.
- **SSL Certificate Generation**: Self-signed certificate generation will need to be implemented using Ansible's openssl modules.
- **Service Coordination**: The FastAPI application depends on PostgreSQL, which will require proper sequencing in Ansible playbooks.
- **Secrets Management**: Redis password and PostgreSQL credentials should be migrated to use Ansible Vault.

### Migration Order

1. **cache role** (Low complexity, foundational service)
   - Implement Memcached configuration
   - Implement Redis configuration with authentication

2. **nginx-multisite role** (Medium complexity, depends on SSL certificates)
   - Implement basic Nginx configuration
   - Implement SSL certificate generation
   - Implement site configuration templates
   - Implement security hardening (fail2ban, UFW)

3. **fastapi-tutorial role** (High complexity, depends on PostgreSQL)
   - Implement PostgreSQL installation and configuration
   - Implement Python environment setup
   - Implement application deployment
   - Implement systemd service configuration

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS/RHEL-based distributions.
2. Self-signed certificates are acceptable for development/testing, but production environments may require integration with proper certificate authorities.
3. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment.
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
5. The Redis password and PostgreSQL credentials in the current implementation are placeholders and will be replaced with proper secrets management.
6. The current Vagrant-based development workflow will be maintained, but with Ansible provisioning instead of Chef.
7. The current implementation does not include monitoring or logging solutions, which may need to be addressed separately.
8. The current implementation assumes a single-server deployment model, which will be maintained in the Ansible implementation.