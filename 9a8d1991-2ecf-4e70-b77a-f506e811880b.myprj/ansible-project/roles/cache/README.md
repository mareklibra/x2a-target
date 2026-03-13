# Cache Role

This Ansible role installs and configures Redis cache server.

## Requirements

- Ansible 2.9 or higher
- Linux distribution (RHEL/CentOS 7/8, Ubuntu 18.04/20.04, Debian 10/11)

## Role Variables

### Main variables

```yaml
# User and group
redis_user: redis
redis_group: redis

# Installation options
redis_package_install: false  # Set to true to install from package
redis_package_name: redis-server
redis_version: "6.2.6"  # Version to install from source

# Directory settings
redis_conf_dir: /etc/redis
redis_data_dir: /var/lib/redis
redis_log_dir: /var/log/redis

# Service settings
redis_disable_os_default: true  # Disable OS default Redis service
redis_enable_service: true  # Enable Redis service
redis_init_system: "systemd"  # Options: systemd, init, upstart
```

### Redis server configuration

```yaml
redis_servers:
  - port: 6379
    address: "0.0.0.0"
    databases: 16
    unixsocket: false
    timeout: 0
    loglevel: "notice"
    rdbcompression: "yes"
    save:
      - "900 1"
      - "300 10"
      - "60 10000"
    maxmemory: "0"
    maxmemory_policy: "volatile-lru"
    appendonly: "no"
    appendfsync: "everysec"
    activerehashing: "yes"
```

## Dependencies

None

## Example Playbook

```yaml
- hosts: cache_servers
  roles:
    - role: cache
      vars:
        redis_servers:
          - port: 6379
            address: "0.0.0.0"
          - port: 6380
            address: "127.0.0.1"
```

## License

Apache-2.0

## Author Information

Ansible Migration Team