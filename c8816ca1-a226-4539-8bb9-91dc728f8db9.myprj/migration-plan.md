# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with caching services (Redis and Memcached) and a FastAPI Python application backed by PostgreSQL. The migration to Ansible will involve converting three primary cookbooks, handling external dependencies, and ensuring security configurations are properly maintained.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- Well-structured Chef cookbooks with clear separation of concerns
- Standard infrastructure components (Nginx, Redis, Memcached, PostgreSQL)
- Security configurations that need careful migration
- Self-signed SSL certificates that need to be managed

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server configured to host multiple SSL-enabled websites with security hardening
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multiple virtual hosts, SSL configuration, security hardening (fail2ban, ufw firewall), system security settings

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - will be replaced by Ansible Galaxy requirements
- `Policyfile.rb` and `Policyfile.lock.json`: Chef policy definitions - will be replaced by Ansible playbooks
- `Vagrantfile`: VM configuration for development/testing - can be adapted for Ansible testing
- `solo.rb` and `solo.json`: Chef Solo configuration - will be replaced by Ansible inventory and variables
- `vagrant-provision.sh`: Provisioning script for Vagrant - will be replaced by Ansible provisioning

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (from Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: Libvirt (from Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or direct package installation
- **ssl_certificate (~> 2.1)**: Replace with Ansible crypto modules for certificate management
- **memcached (~> 6.0)**: Replace with Ansible memcached role or direct package configuration
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or direct package configuration

### Security Considerations

- **SSL Certificate Management**: Self-signed certificates are generated for each site - migrate to Ansible crypto modules
- **Firewall Configuration**: UFW is configured with specific rules - migrate to Ansible ufw or firewalld modules
- **Fail2ban Setup**: Fail2ban is configured for intrusion prevention - migrate to Ansible fail2ban role
- **System Hardening**: Sysctl security settings - migrate to Ansible sysctl module
- **SSH Hardening**: SSH configuration disables root login and password authentication - migrate to Ansible ssh_config module
- **Redis Authentication**: Redis is configured with password authentication - ensure secure password management in Ansible Vault
- **PostgreSQL Security**: Database user and password configuration - ensure secure credential management in Ansible Vault

### Technical Challenges

- **Multi-site Configuration**: The Nginx setup manages multiple virtual hosts with SSL - ensure proper templating in Ansible
- **Service Dependencies**: Ensure proper ordering of service deployments (e.g., PostgreSQL before FastAPI application)
- **SSL Certificate Generation**: Self-signed certificates need to be generated if not existing - implement idempotent certificate creation
- **Security Hardening**: Comprehensive security settings need to be properly migrated - ensure no security regressions
- **Password Management**: Several services use hardcoded passwords - implement Ansible Vault for secure credential management

### Migration Order

1. **Base Infrastructure** (Low complexity)
   - System packages and basic configuration
   - Firewall and security hardening

2. **Cache Services** (Medium complexity)
   - Memcached configuration
   - Redis installation and security setup

3. **Database Layer** (Medium complexity)
   - PostgreSQL installation and configuration
   - Database and user creation

4. **Web Server** (High complexity)
   - Nginx installation
   - SSL certificate generation
   - Virtual host configuration
   - Security settings

5. **Application Deployment** (High complexity)
   - FastAPI application deployment
   - Python environment setup
   - Service configuration

### Assumptions

1. The target environment will continue to be Fedora 42 or compatible Linux distributions
2. Self-signed certificates are acceptable for the migrated environment (not production)
3. The same security posture is required in the Ansible implementation
4. The FastAPI application source will continue to be available at the specified Git repository
5. The multi-site configuration with test.cluster.local, ci.cluster.local, and status.cluster.local will be maintained
6. The current network configuration (192.168.121.10) will be preserved
7. Redis and Memcached configurations will maintain the same performance characteristics
8. PostgreSQL database name, user, and credentials will remain the same

## Ansible Structure Recommendation

```
ansible-nginx-multisite/
├── inventories/
│   ├── development/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
├── roles/
│   ├── common/
│   │   └── # Base system configuration
│   ├── security/
│   │   └── # Security hardening tasks
│   ├── nginx/
│   │   └── # Nginx configuration with multi-site support
│   ├── cache/
│   │   └── # Redis and Memcached configuration
│   ├── database/
│   │   └── # PostgreSQL configuration
│   └── fastapi/
│       └── # FastAPI application deployment
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
├── requirements.yml  # Ansible Galaxy requirements
└── Vagrantfile       # For testing
```

## Implementation Notes

1. **Variable Management**:
   - Move hardcoded values to variables
   - Use Ansible Vault for sensitive information (passwords, keys)

2. **Idempotency**:
   - Ensure all tasks are idempotent, especially certificate generation

3. **Testing Strategy**:
   - Maintain Vagrant setup for testing
   - Implement molecule tests for individual roles

4. **Documentation**:
   - Document each role's purpose and variables
   - Provide examples for different deployment scenarios

5. **Security Considerations**:
   - Implement security best practices for Ansible
   - Ensure no credentials in plain text
   - Maintain or improve current security posture