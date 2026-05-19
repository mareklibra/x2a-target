# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site web application environment with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to Ansible roles and playbooks. The estimated timeline for this migration is 3-4 weeks, with moderate complexity due to the multi-site configuration and security considerations.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and custom configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, UFW firewall)

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

- `Berksfile`: Dependency management for Chef cookbooks. Lists both local and external cookbook dependencies with version constraints. Will be replaced by Ansible Galaxy requirements.yml.
- `solo.json`: Chef run list and node attributes configuration. Will be converted to Ansible group_vars or host_vars.
- `solo.rb`: Chef Solo configuration file. Will be replaced by Ansible configuration.
- `Vagrantfile`: Defines the development VM using Fedora 42. Can be adapted for Ansible testing with minimal changes.
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks in Vagrant. Will be replaced with Ansible provisioner in Vagrant.

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) as specified in cookbook metadata. The Vagrantfile uses Fedora 42.
- **Virtual Machine Technology**: Vagrant with libvirt provider as indicated in the Vagrantfile.
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment.

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package installation and configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package installation and configuration

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Migration should:
  - Maintain the same certificate generation logic or improve with Let's Encrypt integration
  - Preserve the security of private keys with appropriate permissions

- **Firewall Configuration**: UFW firewall is configured with specific rules:
  - Default deny policy
  - Allow SSH, HTTP, and HTTPS
  - Ansible UFW module can directly replace this functionality

- **fail2ban Configuration**: Currently installed and configured with a custom jail.local template:
  - Ansible has fail2ban modules that can replace this functionality

- **SSH Hardening**: SSH configuration includes:
  - Disabling root login
  - Disabling password authentication
  - Ansible's openssh_server module can handle these configurations

- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook recipe
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook
  - FastAPI environment variables contain database credentials
  - Total credentials detected: 3 (Redis password, PostgreSQL user/password)

### Technical Challenges

- **Multi-site Nginx Configuration**: The nginx-multisite cookbook dynamically creates virtual host configurations based on node attributes. Ansible templates will need to replicate this dynamic behavior.
  - Mitigation: Use Ansible template module with Jinja2 loops to generate site configurations from variables.

- **SSL Certificate Generation**: Self-signed certificates are generated for each site. 
  - Mitigation: Use Ansible's openssl_* modules to generate certificates or integrate with certbot for Let's Encrypt certificates.

- **Redis Configuration Hack**: The cache cookbook includes a ruby_block to modify Redis configuration files after they're created.
  - Mitigation: Create a proper Redis configuration template in Ansible rather than modifying files after creation.

- **PostgreSQL User and Database Creation**: The fastapi-tutorial cookbook uses shell commands to create database users and permissions.
  - Mitigation: Use Ansible's postgresql_* modules for more idiomatic database management.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for web services)
   - First migrate the basic Nginx installation and configuration
   - Then add the SSL certificate generation
   - Finally implement the multi-site configuration and security hardening

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Ensure proper integration with Nginx if needed

3. **fastapi-tutorial** (high complexity, application deployment)
   - Set up PostgreSQL database and user creation
   - Implement application deployment from Git
   - Configure Python environment and dependencies
   - Set up systemd service

### Assumptions

1. The current Chef setup assumes a specific directory structure for web content that may need to be preserved in the Ansible migration.
2. Self-signed certificates are acceptable for the environment; no external CA integration is required.
3. The security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment.
4. The Redis password and PostgreSQL credentials in the current setup are development credentials and will be replaced with more secure values in production.
5. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git is accessible and will remain available.
6. The Vagrant development environment will continue to be used for testing the Ansible playbooks.
7. No specific monitoring or logging solutions are integrated that would need to be preserved.