# Migration Plan: cache

**TLDR**: This cookbook installs and configures both Memcached and Redis cache services. It sets up a single Memcached instance with default parameters and a single Redis instance on port 6379 with password authentication. The cookbook handles service installation, configuration, and ensures services are enabled and running.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **memcached**: Default Memcached instance
  - Location/Path: System default (/var/lib/memcached)
  - Port/Socket: 11211 (TCP and UDP)
  - Key Config: 64MB memory, 1024 max connections

- **redis-6379**: Redis instance on port 6379
  - Location/Path: /var/lib/redis
  - Port/Socket: 6379
  - Key Config: Password authentication enabled, 16 databases

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
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.rcinit.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis.upstart.conf.erb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/templates/default/redis@.service.erb
/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/attributes/default.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/attributes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **memcached::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/recipes/default.rb`):
   - Uses custom resource `memcached_instance['memcached']` to install and configure Memcached
   - Sets memory to 64MB, port to 11211, max connections to 1024
   - Configures Memcached to listen on 0.0.0.0
   - Sets ulimit to 1024
   - Resources: memcached_instance (1)

2. **directory creation** (`cookbooks/cache/recipes/default.rb`):
   - Creates directory `/var/log/redis` with owner and group 'redis', mode '0755'
   - Resources: directory (1)

3. **redisio::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/default.rb`):
   - Updates apt package cache
   - Resources: apt_update (1)

4. **redisio::_install_prereqs** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/_install_prereqs.rb`):
   - Installs prerequisite packages based on platform family
   - Resources: package (multiple, platform dependent)

5. **redisio::install** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/install.rb`):
   - Installs Redis from source (version 3.2.11) or package depending on configuration
   - Uses custom resource `redisio_install['redis-installation']` for source installation
   - Resources: build_essential (1), redisio_install (1)

6. **redisio::ulimit** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/ulimit.rb`):
   - Configures ulimit settings for Redis
   - On Debian platforms:
     - Deploys template to `/etc/pam.d/su`
     - Deploys cookbook file to `/etc/pam.d/sudo`
   - Resources: template (1), cookbook_file (1)

7. **redisio::disable_os_default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/disable_os_default.rb`):
   - Stops and disables default OS Redis service if present
   - Resources: service (1) with actions ['stop', 'disable']

8. **redisio::configure** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/configure.rb`):
   - Uses custom resource `redisio_configure['redis-servers']` to configure Redis instances
   - Configures a single Redis instance on port 6379 with password authentication
   - Creates service resources based on init system (systemd, initd, upstart, or rcinit)
   - Resources: redisio_configure (1), service (1)

9. **ruby_block fix_redis_config** (`cookbooks/cache/recipes/default.rb`):
   - Modifies the Redis configuration file at `/etc/redis/6379.conf`
   - Removes specific configuration lines related to replication
   - Resources: ruby_block (1)

10. **redisio::enable** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/enable.rb`):
    - Enables and starts the Redis service
    - Uses the appropriate service name based on init system (redis@6379 for systemd, redis6379 otherwise)
    - Resources: service (1) with actions [:start, :enable]

## Dependencies

**External cookbook dependencies**:
- memcached
- redisio

**System package dependencies**:
- memcached
- build-essential (for Redis compilation if installed from source)
- Redis dependencies (platform specific)

**Service dependencies**:
- memcached.service
- redis@6379.service (systemd) or redis6379 (other init systems)

## Checks for the Migration

**Files to verify**:
- `/etc/redis/6379.conf` - Redis configuration file
- `/var/log/redis` - Redis log directory
- `/var/lib/redis` - Redis data directory
- `/etc/memcached.conf` - Memcached configuration file
- `/var/log/memcached` - Memcached log directory

**Service endpoints to check**:
- Ports listening: 11211 (Memcached TCP/UDP), 6379 (Redis)
- Network interfaces: 0.0.0.0 (both services listen on all interfaces)

**Templates rendered**:
- Redis configuration template renders once for the 6379 instance
- Redis service template renders once based on init system

## Pre-flight checks:
```bash
# Memcached checks
systemctl status memcached
ps aux | grep memcached
netstat -tulpn | grep 11211
ss -tulpn | grep 11211
echo "stats" | nc localhost 11211
echo "version" | nc localhost 11211
memcached-tool localhost:11211 stats

# Redis checks
systemctl status redis@6379  # For systemd
service redis6379 status     # For initd
ps aux | grep redis
netstat -tulpn | grep 6379
ss -tulpn | grep 6379

# Redis connectivity - with authentication
redis-cli -p 6379 -a redis_secure_password_123 ping
redis-cli -p 6379 -a redis_secure_password_123 info server
redis-cli -p 6379 -a redis_secure_password_123 info memory
redis-cli -p 6379 -a redis_secure_password_123 info keyspace

# Configuration validation
cat /etc/redis/6379.conf | grep -E 'port|requirepass|bind'
cat /etc/redis/6379.conf | grep -v "^replica-serve-stale-data"  # Should not be present
cat /etc/redis/6379.conf | grep -v "^replica-read-only"         # Should not be present
cat /etc/redis/6379.conf | grep -v "^repl-ping-replica-period"  # Should not be present
cat /etc/redis/6379.conf | grep -v "^client-output-buffer-limit"  # Should not be present
cat /etc/redis/6379.conf | grep -v "^replica-priority"          # Should not be present

# Memcached configuration
cat /etc/memcached.conf | grep -E 'memory|port|listen|maxconn'

# Log files
tail -f /var/log/redis/redis_6379.log
tail -f /var/log/memcached.log

# Directory permissions
ls -la /var/log/redis
ls -la /var/lib/redis
ls -la /var/lib/memcached

# Resource usage
ps aux | grep redis-server | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
ps aux | grep memcached | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
```