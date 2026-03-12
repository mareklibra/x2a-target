# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up a single Memcached instance with default configuration and a single Redis instance on port 6379 with password authentication. The cookbook handles installation, configuration, and service management for both services.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **memcached**: Default Memcached instance
  - Location/Path: System default (/etc/memcached.conf)
  - Port/Socket: 11211 (TCP and UDP)
  - Key Config: 64MB memory, 1024 max connections, listening on 0.0.0.0

- **redis-6379**: Redis instance with authentication
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Password authentication enabled with 'redis_secure_password_123'

## File Structure

```
cookbooks/cache/recipes/default.rb
/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/recipes/default.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/default.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/_install_prereqs.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/install.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/ulimit.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/disable_os_default.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/configure.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/enable.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/providers/configure.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/providers/install.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.conf.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.init.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.upstart.conf.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis@.service.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.rcinit.erb
/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/attributes/default.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/attributes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes memcached recipe
   - Sets Redis server configuration with authentication
   - Creates Redis log directory
   - Includes Redis recipes
   - Fixes Redis configuration with a ruby_block
   - Resources: include_recipe (3), directory (1), ruby_block (1)

2. **memcached::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/recipes/default.rb`):
   - Uses custom resource: memcached_instance['memcached']
   - Configures a single Memcached instance with attributes:
     - memory: 64MB
     - port: 11211
     - udp_port: 11211
     - listen: 0.0.0.0
     - maxconn: 1024
     - max_object_size: 1m
   - Resources: memcached_instance (1)

3. **redisio::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/default.rb`):
   - Updates apt repositories
   - Includes _install_prereqs recipe if not using package install
   - Includes install recipe if not bypassing setup
   - Resources: apt_update (1), include_recipe (2)

4. **redisio::_install_prereqs** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/_install_prereqs.rb`):
   - Installs prerequisite packages based on platform
   - Resources: package (multiple, platform dependent)

5. **redisio::install** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/install.rb`):
   - Installs Redis either from package or source
   - If package install: installs redis-server package
   - If source install: includes _install_prereqs and uses redisio_install custom resource
   - Includes ulimit recipe
   - Resources: package (1) or redisio_install (1), include_recipe (1)

6. **redisio::ulimit** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/ulimit.rb`):
   - Configures ulimit settings for Redis
   - On Debian systems:
     - Configures /etc/pam.d/su
     - Configures /etc/pam.d/sudo
   - Sets user-specific ulimit settings if defined
   - Resources: template (1), cookbook_file (1), user_ulimit (conditional)

7. **redisio::disable_os_default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/disable_os_default.rb`):
   - Stops and disables default OS Redis service if present
   - Resources: service (1)

8. **redisio::configure** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/configure.rb`):
   - Includes default and ulimit recipes
   - Uses redisio_configure custom resource to configure Redis servers
   - Iterates through Redis instances to set up services based on job control system:
     - For instance: 6379
       - If systemd: Sets up redis@6379 service
       - If initd: Sets up redis6379 service
       - If upstart: Sets up redis6379 service
       - If rcinit: Sets up redis6379 service
   - Resources: redisio_configure (1), service (1 per instance)

9. **redisio::enable** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/enable.rb`):
   - Iterates through Redis servers to start and enable services
   - For instance: 6379
     - Enables and starts redis@6379 (systemd) or redis6379 (other init systems)
   - Resources: service (1 per instance)

10. **ruby_block[fix_redis_config]** (`cookbooks/cache/recipes/default.rb`):
    - Modifies Redis configuration file after it's created
    - Removes specific configuration lines related to replication
    - File modified: /etc/redis/6379.conf
    - Resources: ruby_block (1)

## Dependencies

**External cookbook dependencies**: 
- memcached (~> 6.0)
- redisio

**System package dependencies**: 
- memcached
- redis-server (if package install is used)
- build-essential (if source install is used)

**Service dependencies**: 
- memcached.service
- redis@6379.service (systemd) or redis6379 (other init systems)

## Checks for the Migration

**Files to verify**:
- /etc/memcached.conf
- /etc/redis/6379.conf
- /var/log/redis/
- /var/lib/redis/
- /var/run/redis/
- /etc/systemd/system/redis@6379.service (if systemd)
- /etc/init.d/redis6379 (if initd)
- /etc/init/redis6379.conf (if upstart)

**Service endpoints to check**:
- Ports listening: 
  - 11211 (TCP/UDP) - Memcached
  - 6379 (TCP) - Redis
- Unix sockets: None specified in the configuration
- Network interfaces: 0.0.0.0 (both services listen on all interfaces)

**Templates rendered**:
- redis.conf.erb → /etc/redis/6379.conf (1 instance)
- redis@.service.erb → /lib/systemd/system/redis@6379.service (if systemd, 1 instance)
- redis.init.erb → /etc/init.d/redis6379 (if initd, 1 instance)
- redis.upstart.conf.erb → /etc/init/redis6379.conf (if upstart, 1 instance)
- redis.rcinit.erb → /usr/local/etc/rc.d/redis6379 (if rcinit, 1 instance)

## Pre-flight checks:
```bash
# Memcached checks
systemctl status memcached
ps aux | grep memcached
netstat -tulpn | grep 11211
ss -tulpn | grep 11211
echo "stats" | nc localhost 11211
echo "version" | nc localhost 11211

# Redis checks
# Service status
systemctl status redis@6379 || service redis6379 status
ps aux | grep redis-server

# Redis connectivity
redis-cli -h localhost -p 6379 ping
redis-cli -h localhost -p 6379 -a redis_secure_password_123 ping
redis-cli -h localhost -p 6379 -a redis_secure_password_123 info server
redis-cli -h localhost -p 6379 -a redis_secure_password_123 info memory

# Configuration validation
cat /etc/redis/6379.conf | grep -E 'port|requirepass|bind'
cat /etc/redis/6379.conf | grep -v '^replica-serve-stale-data'
cat /etc/redis/6379.conf | grep -v '^replica-read-only'
cat /etc/redis/6379.conf | grep -v '^repl-ping-replica-period'
cat /etc/redis/6379.conf | grep -v '^client-output-buffer-limit'
cat /etc/redis/6379.conf | grep -v '^replica-priority'

# Logs
tail -f /var/log/redis/redis-server.log
journalctl -u redis@6379 -f

# Network listening
netstat -tulpn | grep 6379
ss -tulpn | grep redis
lsof -i :6379

# Data directories
ls -lah /var/lib/redis/
ls -lah /var/run/redis/
df -h /var/lib/redis/
```