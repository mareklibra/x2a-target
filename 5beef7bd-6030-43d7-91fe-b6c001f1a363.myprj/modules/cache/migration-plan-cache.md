# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up a single Memcached instance with default settings and one Redis instance on port 6379 with password authentication. The cookbook handles installation, configuration, and service management for both caching solutions.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **memcached**: Default Memcached instance
  - Location/Path: System default (/etc/memcached.conf)
  - Port/Socket: 11211 (TCP and UDP)
  - Key Config: 64MB memory, 1024 max connections, listens on 0.0.0.0

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
   - Configures a single Memcached instance with default settings:
     - Memory: 64MB
     - Port: 11211 (TCP and UDP)
     - Listen: 0.0.0.0
     - Max connections: 1024
     - Max object size: 1MB
   - Resources: memcached_instance (1)

3. **redisio::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/default.rb`):
   - Updates apt repositories
   - Includes prerequisite installation recipes
   - Resources: apt_update (1), include_recipe (1)

4. **redisio::_install_prereqs** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/_install_prereqs.rb`):
   - Installs required packages for Redis
   - Resources: package (multiple), build_essential (1)

5. **redisio::install** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/install.rb`):
   - Installs Redis either from package or source based on configuration
   - Uses custom resource: redisio_install['redis-installation']
   - Includes ulimit recipe for system limits configuration
   - Resources: package (1) or redisio_install (1), build_essential (1)

6. **redisio::ulimit** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/ulimit.rb`):
   - Configures system limits for Redis
   - On Debian systems:
     - Template: /etc/pam.d/su
     - Cookbook file: /etc/pam.d/sudo
   - Sets user limits for Redis user
   - Resources: template (1), cookbook_file (1), user_ulimit (1)

7. **redisio::disable_os_default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/disable_os_default.rb`):
   - Stops and disables default OS Redis service if present
   - Resources: service (1)

8. **redisio::configure** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/configure.rb`):
   - Uses custom resource: redisio_configure['redis-servers']
   - Provider: /workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/providers/configure.rb
   - For the Redis server on port 6379:
     - Creates Redis user and group
     - Creates configuration directories
     - Creates data directories
     - Creates PID directories
     - Creates log directories
     - Configures SELinux if enabled
     - Renders Redis configuration file
     - Sets up init scripts based on job control system (systemd, initd, upstart, or rcinit)
   - Resources: redisio_configure (1), service (1)

9. **ruby_block[fix_redis_config]** (`cookbooks/cache/recipes/default.rb`):
   - Modifies Redis configuration to remove certain replication settings
   - Removes lines related to replica configuration
   - Resources: ruby_block (1)

10. **redisio::enable** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/enable.rb`):
    - Starts and enables Redis service for each configured server
    - For the Redis instance on port 6379:
      - If systemd: service[redis@6379]
      - Otherwise: service[redis6379]
    - Resources: service (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- memcached
- redis-server (if package install is used)
- build-essential (if compiling from source)

**Service dependencies**:
- memcached.service
- redis@6379.service (systemd) or redis6379 (initd/upstart)

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
  - 11211 (Memcached TCP and UDP)
  - 6379 (Redis)
- Unix sockets: None specified in default configuration
- Network interfaces: 0.0.0.0 (both services listen on all interfaces)

**Templates rendered**:
- redis.conf.erb → /etc/redis/6379.conf (1 instance)
- redis@.service.erb → /etc/systemd/system/redis@6379.service (if systemd)
- redis.init.erb → /etc/init.d/redis6379 (if initd)
- redis.upstart.conf.erb → /etc/init/redis6379.conf (if upstart)
- redis.rcinit.erb → /usr/local/etc/rc.d/redis6379 (if FreeBSD)

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
systemctl status redis@6379  # For systemd
service redis6379 status     # For initd

# Process verification
ps aux | grep redis
pgrep -f redis-server

# Redis connectivity - with authentication
redis-cli -p 6379 -a redis_secure_password_123 ping
redis-cli -p 6379 -a redis_secure_password_123 info server
redis-cli -p 6379 -a redis_secure_password_123 info clients
redis-cli -p 6379 -a redis_secure_password_123 info memory

# Configuration validation
cat /etc/redis/6379.conf | grep -E 'port|requirepass|bind'
cat /etc/redis/6379.conf | grep -v '^#' | grep -v '^$'  # Show non-comment, non-empty lines

# Verify removed configuration lines
cat /etc/redis/6379.conf | grep -E 'replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority'
# Should return nothing if ruby_block fix worked correctly

# Network listening
netstat -tulpn | grep 6379
ss -tulpn | grep 6379
lsof -i :6379

# Logs
tail -f /var/log/redis/redis-server.log  # Actual path may vary based on configuration
journalctl -u redis@6379 -f  # For systemd

# Memory usage
ps -o pid,user,%mem,rss,vsz -p $(pgrep -f redis-server)
ps -o pid,user,%mem,rss,vsz -p $(pgrep -f memcached)

# File permissions
ls -la /etc/redis/
ls -la /var/lib/redis/
ls -la /var/run/redis/
ls -la /var/log/redis/

# Authentication test
echo "AUTH redis_secure_password_123" | nc localhost 6379
echo "AUTH redis_secure_password_123\r\nPING" | nc localhost 6379
```