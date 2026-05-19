# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure setup for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting three Chef cookbooks with their recipes, templates, attributes, and custom resources to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible Roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has well-structured Chef cookbooks with clear separation of concerns
- No complex custom resources (only one simple lineinfile resource)
- Standard infrastructure components (Nginx, Redis, Memcached, PostgreSQL)
- Security configurations that need careful migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and firewall configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw), sysctl security settings

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

- `Berksfile`: Defines cookbook dependencies including nginx (~> 12.0), memcached (~> 6.0), and redisio (~> 7.2.4)
- `solo.json`: Contains the run list and configuration attributes for Nginx sites and security settings
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Defines a Fedora 42 VM with port forwarding and resource allocation
- `vagrant-provision.sh`: Bash script to install Chef and run the cookbooks in the Vagrant environment

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified (appears to be a local development environment)

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached` or custom Ansible tasks
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or custom Ansible tasks
- **Python 3 and venv**: Use Ansible's `pip` and `package` modules to install Python dependencies
- **PostgreSQL**: Use Ansible's `postgresql_*` modules or `geerlingguy.postgresql` role

### Security Considerations

- **SSL Certificate Management**: 
  - Chef generates self-signed certificates for each site
  - Migration approach: Use Ansible's `openssl_*` modules to generate certificates or integrate with Let's Encrypt using `geerlingguy.certbot`

- **Firewall Configuration**: 
  - Chef configures UFW with specific rules
  - Migration approach: Use Ansible's `ufw` module to configure identical rules

- **fail2ban Configuration**: 
  - Chef installs and configures fail2ban
  - Migration approach: Use Ansible's `template` module to create fail2ban configuration files

- **SSH Hardening**: 
  - Chef disables root login and password authentication
  - Migration approach: Use Ansible's `lineinfile` module or `ansible.posix.sshd` module to configure SSH

- **Vault/secrets management**:
  - Redis password in `cookbooks/cache/recipes/default.rb` (hardcoded as 'redis_secure_password_123')
  - PostgreSQL credentials in `cookbooks/fastapi-tutorial/recipes/default.rb` (hardcoded as 'fastapi_password')
  - SSL certificates and private keys (generated during deployment)
  - Migration approach: Use Ansible Vault to store sensitive information

### Technical Challenges

- **Custom Resource Migration**: 
  - The `lineinfile` custom resource in nginx-multisite needs to be replaced with Ansible's native `lineinfile` module
  - Complexity: Low

- **Template Conversion**: 
  - Converting ERB templates to Jinja2 format for Ansible
  - Complexity: Medium
  - Approach: Carefully map ERB syntax (`<%= %>`) to Jinja2 syntax (`{{ }}`)

- **Attribute to Variable Mapping**: 
  - Chef attributes need to be converted to Ansible variables
  - Complexity: Medium
  - Approach: Create group_vars and host_vars files to store configuration data

- **Idempotency Verification**: 
  - Ensure all Ansible tasks are idempotent, especially for the custom logic in recipes
  - Complexity: Medium
  - Approach: Use Ansible's built-in idempotency features and test thoroughly

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Contains security configurations that should be established first

2. **cache** (Priority 2)
   - Depends on external cookbooks (memcached, redisio)
   - Relatively simple configuration

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on properly configured infrastructure
   - More complex with database setup, Python environment, and application deployment

### Assumptions

1. The target environment will continue to be Fedora-based systems (specifically Fedora 42 as specified in the Vagrantfile)
2. Self-signed certificates are acceptable for the migrated solution (as used in the current Chef implementation)
3. The same security hardening measures should be implemented in the Ansible solution
4. The directory structure for web content will remain the same
5. The PostgreSQL database schema and user permissions will remain unchanged
6. The FastAPI application source will continue to be pulled from the same Git repository
7. The Redis password and PostgreSQL credentials will be managed securely in the new implementation
8. The Vagrant development environment will be maintained for testing the Ansible playbooks