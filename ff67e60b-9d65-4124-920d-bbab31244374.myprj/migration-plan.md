# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting 3 Chef cookbooks with their recipes, templates, and attributes to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- **Total**: 5-7 weeks

**Complexity**: Medium
- The codebase is well-structured with clear separation of concerns
- Security configurations are comprehensive but straightforward
- No complex custom resources or libraries are used

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server configured to host multiple SSL-enabled websites with security hardening
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw firewall), system security settings

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements file
- `Policyfile.rb` and `Policyfile.lock.json`: Chef policy definition - will be replaced by Ansible playbook structure
- `Vagrantfile`: VM configuration for development/testing - can be adapted for Ansible testing
- `solo.rb`: Chef Solo configuration - will be replaced by Ansible configuration
- `solo.json`: Chef node attributes - will be converted to Ansible variables
- `vagrant-provision.sh`: Provisioning script - will be replaced by Ansible provisioning

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (from Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Libvirt (from Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package configuration
- **ssl_certificate (~> 2.1)**: Replace with Ansible certificate management tasks

### Security Considerations

- **SSL/TLS Configuration**: The current setup generates self-signed certificates. Ansible should maintain this functionality while providing a path to use Let's Encrypt or other certificate authorities.
- **Firewall (ufw)**: Convert ufw configuration to Ansible firewall module tasks
- **fail2ban**: Convert fail2ban configuration to Ansible tasks
- **System Hardening**: Convert sysctl security settings to Ansible sysctl module
- **SSH Hardening**: Convert SSH security configurations to Ansible ssh_config module
- **Secrets Management**: 
  - Redis password is hardcoded in Chef recipe
  - PostgreSQL credentials are hardcoded in FastAPI recipe
  - These should be moved to Ansible Vault or another secrets management solution

### Technical Challenges

- **Multi-site Configuration**: Ensure the dynamic generation of Nginx site configurations is properly implemented in Ansible
- **SSL Certificate Management**: Implement proper certificate generation and management in Ansible
- **Service Dependencies**: Maintain proper ordering of service installation, configuration, and startup
- **Idempotency**: Ensure all operations remain idempotent, particularly database user creation and application deployment

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Start with basic Nginx installation and configuration
   - Add SSL certificate management
   - Add security hardening components
   - Implement multi-site configuration

2. **cache** (Priority 2)
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Ensure proper service management

3. **fastapi-tutorial** (Priority 3)
   - Set up PostgreSQL database
   - Configure Python environment
   - Deploy application from Git
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora/RHEL-based systems, with potential for Ubuntu/Debian support
2. Self-signed certificates are acceptable for development, but production deployment may require integration with Let's Encrypt or similar
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain available
4. The current security settings are appropriate for the target environment
5. No custom Chef resources or libraries are being used that would require special handling
6. The current VM resources (2GB RAM, 2 CPUs) are sufficient for the application stack

## Ansible Structure Recommendation

```
ansible-nginx-multisite/
├── inventories/
│   ├── development/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
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
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
├── requirements.yml
└── vagrant.yml
```

## Testing Strategy

1. Develop a parallel Vagrant environment for Ansible testing
2. Create test cases for each role:
   - Nginx site configuration and accessibility
   - SSL certificate generation and validation
   - Security configuration verification
   - Cache service functionality
   - FastAPI application deployment and functionality
3. Implement CI/CD pipeline for automated testing
4. Perform side-by-side comparison with Chef-managed environment

## Knowledge Transfer Plan

1. Document each Ansible role with detailed README files
2. Create a migration guide for team members
3. Conduct knowledge sharing sessions on:
   - Ansible basics for Chef users
   - New repository structure
   - Role customization
   - Variable management
   - Secrets handling
4. Provide hands-on training with the new Ansible environment