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
- External dependencies on community cookbooks need to be replaced with Ansible Galaxy roles
- Security configurations and SSL certificate management require careful handling
- Secrets management needs to be implemented with Ansible Vault

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and site configuration
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
    - Key Features: Git repository deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external dependencies with version constraints. Will be replaced by Ansible Galaxy requirements.yml.
- `solo.json`: Chef configuration file containing the run list and node attributes. Will be replaced by Ansible inventory variables.
- `solo.rb`: Chef configuration file specifying cookbook paths and log settings. Will be replaced by Ansible configuration.
- `Vagrantfile`: Defines the development VM configuration using Vagrant. Can be adapted for Ansible by changing the provisioner.
- `vagrant-provision.sh`: Shell script to install Chef and run the cookbooks in the Vagrant environment. Will be replaced by Ansible provisioning.

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ mentioned in cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` module or community.general collection
- **memcached (~> 6.0)**: Replace with Ansible's `memcached` module or geerlingguy.memcached role
- **redisio (~> 7.2.4)**: Replace with Ansible's `redis` module or geerlingguy.redis role
- **Python packages**: Use Ansible's `pip` module to install requirements

### Security Considerations

- **SSL Certificate Management**: 
  - The current implementation generates self-signed certificates
  - Migration approach: Use Ansible's `openssl_*` modules for certificate generation or community.crypto collection

- **Firewall Configuration**: 
  - Current implementation uses UFW
  - Migration approach: Use Ansible's `ufw` module or `firewalld` module depending on target OS

- **Fail2ban Configuration**: 
  - Current implementation installs and configures fail2ban
  - Migration approach: Use Ansible to deploy fail2ban configuration templates

- **SSH Hardening**: 
  - Current implementation disables root login and password authentication
  - Migration approach: Use Ansible's `lineinfile` module or dedicated ssh hardening role

- **Vault/secrets management**:
  - Redis password is hardcoded in the recipe
  - PostgreSQL credentials are hardcoded in the recipe
  - Migration approach: Move all credentials to Ansible Vault

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation dynamically generates site configurations based on node attributes
  - Mitigation: Use Ansible templates with loops over host variables to achieve the same dynamic configuration

- **SSL Certificate Generation**: 
  - Description: Self-signed certificates are generated for each site
  - Mitigation: Use Ansible's openssl modules to generate certificates or integrate with Let's Encrypt

- **Service Dependencies**: 
  - Description: The FastAPI application depends on PostgreSQL being configured first
  - Mitigation: Use Ansible handlers and proper task ordering to ensure dependencies are met

- **Redis Configuration Hacks**: 
  - Description: The current implementation includes a hack to fix Redis configuration
  - Mitigation: Create proper Redis configuration templates in Ansible

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Includes security hardening that should be applied first

2. **cache** (Priority 2)
   - Provides caching services that the application may depend on
   - Moderate complexity with Redis authentication

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on other infrastructure components
   - Includes database setup and application configuration

### Assumptions

1. The target environment will continue to be Fedora-based, though the cookbooks support Ubuntu and CentOS as well
2. Self-signed certificates are acceptable for development/testing environments
3. The current security configurations (fail2ban, UFW, SSH hardening) are appropriate for the target environment
4. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
5. The current Redis and PostgreSQL passwords are development/testing credentials and will be replaced in production
6. The Vagrant development environment will continue to be used for testing

## Ansible Structure Recommendation

```
ansible-nginx-multisite/
├── inventories/
│   ├── development/
│   │   ├── group_vars/
│   │   │   └── all.yml  # Variables from solo.json
│   │   └── hosts        # Development hosts
│   └── production/
│       ├── group_vars/
│       │   └── all.yml  # Production variables
│       └── hosts        # Production hosts
├── roles/
│   ├── nginx-multisite/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── tasks/
│   │   └── templates/
│   ├── cache/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── tasks/
│   │   └── templates/
│   └── fastapi-tutorial/
│       ├── defaults/
│       ├── handlers/
│       ├── tasks/
│       └── templates/
├── playbooks/
│   ├── site.yml         # Main playbook (equivalent to run_list)
│   ├── nginx.yml        # Nginx-specific playbook
│   ├── cache.yml        # Cache-specific playbook
│   └── fastapi.yml      # FastAPI-specific playbook
├── requirements.yml     # Ansible Galaxy requirements (replacing Berksfile)
└── Vagrantfile          # Updated to use Ansible provisioner
```

## Detailed Migration Tasks

1. **Infrastructure Setup**
   - Create Ansible directory structure
   - Set up Ansible configuration
   - Create requirements.yml for external dependencies
   - Update Vagrantfile to use Ansible provisioner

2. **Variable Migration**
   - Convert node attributes from solo.json to Ansible variables
   - Create vault.yml for sensitive information
   - Define environment-specific variables

3. **Role Development**
   - Create Ansible roles corresponding to each Chef cookbook
   - Convert Chef recipes to Ansible tasks
   - Convert Chef templates to Ansible templates
   - Implement handlers for service notifications

4. **Testing**
   - Test each role individually
   - Test the complete playbook
   - Verify idempotence
   - Compare results with Chef-provisioned environment

5. **Documentation**
   - Document each role
   - Create README with usage instructions
   - Document variables and their purposes
   - Create example inventory