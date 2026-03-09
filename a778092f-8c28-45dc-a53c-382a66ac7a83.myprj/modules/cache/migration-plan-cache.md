# Migration Plan: cache

**TLDR**: This cookbook installs and configures two caching services: Memcached and Redis. It sets up a single Memcached instance with default settings and one Redis instance on port 6379 with password authentication. The cookbook handles installation, configuration, and service management for both caching solutions.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **memcached**: Default Memcached instance
  - Location/Path: System default (/var/lib/memcached)
  - Port/Socket: 11211 (TCP and UDP)
  - Key Config: 64MB memory, 1024 max connections

- **redis-6379**: Redis instance with authentication
  - Location/Path: /var/lib/redis
  - Port/Socket: 6379
  - Key Config: Password authentication enabled, 16 databases

## File Structure

**Recipes:**
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
```

**Providers:**
```
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/providers/configure.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/providers/install.rb
```

**Templates:**
```
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.conf.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.init.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.rcinit.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.upstart.conf.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis@.service.erb
```

**Attributes:**
```
/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/attributes/default.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/attributes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **cache::default** (`cookbooks/cache/recipes/default.rb`):
   - Includes memcached recipe
   - Sets Redis server configuration with password authentication
   - Creates Redis log directory
   - Includes Redis recipes
   - Modifies Redis configuration file to remove specific settings
   - Resources: include_recipe (3), directory (1), ruby_block (1)

2. **memcached::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/recipes/default.rb`):
   - Uses custom resource: memcached_instance['memcached']
   - Configures Memcached with 64MB memory, port 11211, and 1024 max connections
   - Resources: memcached_instance (1)

3. **redisio::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/default.rb`):
   - Updates apt repositories
   - Includes installation prerequisites
   - Includes Redis installation recipe
   - Resources: apt_update (1), include_recipe (2)

4. **redisio::_install_prereqs** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/_install_prereqs.rb`):
   - Installs platform-specific packages required for Redis
   - Resources: package (multiple, platform-dependent)

5. **redisio::install** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/install.rb`):
   - Installs Redis either from package or source based on configuration
   - If package install: Installs redis-server package
   - If source install: Compiles Redis from source
   - Includes ulimit recipe for system limits configuration
   - Resources: package (1) or redisio_install (1), build_essential (1)

6. **redisio::ulimit** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/ulimit.rb`):
   - Configures system limits for Redis user
   - On Debian systems, modifies PAM configuration
   - Resources: template (1), cookbook_file (1), user_ulimit (conditional)

7. **redisio::disable_os_default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/disable_os_default.rb`):
   - Stops and disables default OS Redis service if present
   - Resources: service (1)

8. **redisio::configure** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/configure.rb`):
   - Configures Redis server instances
   - Uses custom resource: redisio_configure['redis-servers']
   - Creates service resources for each Redis instance
   - Conditional service setup based on init system (systemd, upstart, initd, or rcinit)
   - Resources: redisio_configure (1), service (1 per instance)

9. **ruby_block[fix_redis_config]** (`cookbooks/cache/recipes/default.rb`):
   - Modifies Redis configuration file to remove specific replication settings
   - Removes lines related to replica configuration
   - Resources: ruby_block (1)

10. **redisio::enable** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/enable.rb`):
    - Starts and enables Redis services for each configured instance
    - For this cookbook, only one instance (port 6379)
    - Resources: service (1)

## Dependencies

**External cookbook dependencies**:
- memcached
- redisio

**System package dependencies**:
- memcached
- redis-server (if package install)
- build-essential (if source install)

**Service dependencies**:
- memcached.service
- redis@6379.service (systemd) or redis6379 (initd)

## Checks for the Migration

**Files to verify**:
- /etc/memcached.conf
- /etc/redis/6379.conf
- /var/log/redis/
- /var/lib/redis/
- /etc/systemd/system/redis@.service (if systemd)
- /etc/init.d/redis6379 (if initd)
- /etc/init/redis6379.conf (if upstart)

**Service endpoints to check**:
- Ports listening:
  - 11211 (Memcached TCP and UDP)
  - 6379 (Redis)
- Network interfaces: 0.0.0.0 (Memcached), default Redis binding

**Templates rendered**:
- redis.conf.erb → /etc/redis/6379.conf
- redis@.service.erb → /etc/systemd/system/redis@.service (if systemd)
- redis.init.erb → /etc/init.d/redis6379 (if initd)
- redis.upstart.conf.erb → /etc/init/redis6379.conf (if upstart)
- redis.rcinit.erb → /etc/rc.d/redis6379 (if rcinit)

## Pre-flight checks:

```bash
# Memcached checks
systemctl status memcached
ps aux | grep memcached
netstat -tulpn | grep 11211
ss -tulpn | grep 11211
echo "stats" | nc localhost 11211
echo "version" | nc localhost 11211
cat /etc/memcached.conf

# Redis checks
systemctl status redis@6379  # if systemd
service redis6379 status     # if initd
ps aux | grep redis
netstat -tulpn | grep 6379
ss -tulpn | grep 6379

# Redis connectivity test
redis-cli -h localhost -p 6379 ping
redis-cli -h localhost -p 6379 -a redis_secure_password_123 ping
redis-cli -h localhost -p 6379 -a redis_secure_password_123 info server
redis-cli -h localhost -p 6379 -a redis_secure_password_123 info memory

# Redis configuration check
cat /etc/redis/6379.conf | grep -E 'port|requirepass|bind'
cat /etc/redis/6379.conf | grep -v '^#' | grep -v '^$'  # Show non-comment, non-empty lines

# Check for removed configuration lines
cat /etc/redis/6379.conf | grep -E 'replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority'

# Directory permissions
ls -la /var/log/redis/
ls -la /var/lib/redis/

# Service files
if [ -f /etc/systemd/system/redis@.service ]; then
  cat /etc/systemd/system/redis@.service
elif [ -f /etc/init.d/redis6379 ]; then
  cat /etc/init.d/redis6379
elif [ -f /etc/init/redis6379.conf ]; then
  cat /etc/init/redis6379.conf
fi

# Resource usage
ps aux | grep redis | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
ps aux | grep memcached | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
```