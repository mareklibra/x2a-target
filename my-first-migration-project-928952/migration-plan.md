# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Memcached and Redis) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies on community cookbooks need to be replaced with Ansible equivalents
- Security configurations and SSL certificate management require careful handling
- Secrets management needs to be improved during migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw), custom nginx configurations

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Memcached configuration, Redis with password authentication, service management

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external dependencies with version constraints. Will be replaced by Ansible requirements.yml.
- `solo.json`: Chef configuration file containing the run list and node attributes. Will be replaced by Ansible inventory variables.
- `solo.rb`: Chef configuration file specifying paths and log settings. Will be replaced by Ansible configuration.
- `Vagrantfile`: Defines the development VM using Fedora 42. Can be adapted for Ansible testing with minimal changes.
- `vagrant-provision.sh`: Shell script to install Chef and run the provisioning process. Will be replaced with Ansible provisioning commands.

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) as specified in cookbook metadata. The Vagrantfile uses Fedora 42 for development.
- **Virtual Machine Technology**: Vagrant with libvirt provider as indicated in the Vagrantfile.
- **Cloud Platform**: No specific cloud platform configurations detected. The setup appears to be designed for on-premises or generic VM deployment.

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible's `geerlingguy.memcached` role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible's `geerlingguy.redis` role or direct package installation and configuration

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migration should:
  - Maintain the same certificate generation capability for development
  - Add support for production certificates (Let's Encrypt integration)
  - Ensure proper permissions on private keys

- **Firewall Configuration**: The current implementation uses UFW. Migration should:
  - Use appropriate firewall modules based on the target OS (ufw for Ubuntu, firewalld for CentOS/Fedora)
  - Maintain the same rule set (SSH, HTTP, HTTPS)

- **Fail2ban Configuration**: Maintain the same protection against brute force attacks

- **SSH Hardening**: Maintain the same security settings:
  - Disable root login
  - Disable password authentication

- **Vault/secrets management**:
  - Redis password in cache cookbook (hardcoded as 'redis_secure_password_123')
  - PostgreSQL database credentials in fastapi-tutorial cookbook (hardcoded as 'fastapi_password')
  - Environment variables in .env file for FastAPI application
  - Total credentials detected: 2 hardcoded passwords, 1 environment file with sensitive data

### Technical Challenges

- **Multi-site Nginx Configuration**: The current implementation uses templates to generate site configurations. Ansible will need to:
  - Generate similar configuration files from templates
  - Handle the dynamic nature of site definitions
  - Ensure proper reloading of Nginx when configurations change

- **Redis Configuration Hack**: The current implementation includes a Ruby block to modify Redis configuration files after they're created. Ansible will need:
  - A cleaner approach to Redis configuration, possibly using templates or lineinfile modules
  - Proper validation to ensure configuration is correct

- **PostgreSQL User and Database Creation**: The current implementation uses shell commands. Ansible should:
  - Use the postgresql_user and postgresql_db modules for cleaner, idempotent operations
  - Handle password management securely

- **Service Management**: Ensure proper ordering of service installation, configuration, and startup

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Relatively self-contained with clear configuration patterns

2. **cache** (Priority 2)
   - Depends on external services (memcached, redis)
   - Moderate complexity with authentication requirements

3. **fastapi-tutorial** (Priority 3)
   - Most complex with multiple dependencies (PostgreSQL, Python environment)
   - Application deployment that depends on infrastructure being in place

### Assumptions

1. The target environment will continue to support both Ubuntu and CentOS/RHEL as specified in the cookbook metadata.
2. The Vagrant development environment will be maintained for testing.
3. Self-signed certificates are acceptable for development, but production deployment may require proper certificates.
4. The current security settings (fail2ban, ufw, SSH hardening) are appropriate for the target environment.
5. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available.
6. The hardcoded passwords in the current implementation will be replaced with Ansible Vault or another secure method.
7. The current directory structure and naming conventions will be maintained where possible.
8. The migration will not involve significant changes to the application architecture or deployment strategy.