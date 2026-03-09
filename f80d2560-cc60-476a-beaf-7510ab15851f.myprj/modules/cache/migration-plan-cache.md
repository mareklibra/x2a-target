# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up a single Memcached instance with default configuration and a single Redis instance on port 6379 with password authentication. The cookbook handles installation, configuration, and service management for both services.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **memcached**: Default Memcached instance
  - Location/Path: System default (/etc/memcached.conf on Debian)
  - Port/Socket: 11211 (TCP and UDP)
  - Key Config: 64MB memory, 1024 max connections

- **redis-6379**: Redis instance with authentication
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: requirepass='redis_secure_password_123', 16 databases

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
/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/attributes/default.rb
/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/attributes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **memcached::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/memcached-7992788f1a376defb902059063f5295e37d281cb/recipes/default.rb`):
   - Uses custom resource: memcached_instance['memcached']
   - Configures Memcached with the following settings:
     - memory: 64MB
     - port: 11211
     - udp_port: 11211
     - listen: 0.0.0.0
     - maxconn: 1024
     - max_object_size: 1m
   - Resources: memcached_instance (1)

2. **directory** (`cookbooks/cache/recipes/default.rb`):
   - Creates directory: /var/log/redis
   - Sets owner: redis
   - Sets group: redis
   - Sets mode: 0755
   - Sets recursive: true
   - Resources: directory (1)

3. **redisio::default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/default.rb`):
   - Updates apt repositories
   - Includes redisio::_install_prereqs if not using package install
     - Installs prerequisite packages based on platform
   - Resources: apt_update (1), package (multiple), build_essential (1)

4. **redisio::install** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/install.rb`):
   - Installs Redis either from package or source based on node['redisio']['package_install']
   - If using package: Installs redis-server package
   - If using source: Uses custom resource redisio_install['redis-installation']
   - Includes redisio::ulimit to configure system limits
   - Resources: package (1) or redisio_install (1), build_essential (1)

5. **redisio::ulimit** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/ulimit.rb`):
   - On Debian platforms:
     - Configures /etc/pam.d/su
     - Configures /etc/pam.d/sudo
   - Sets up user limits if defined
   - Resources: template (1), cookbook_file (1), user_ulimit (conditional)

6. **redisio::disable_os_default** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/disable_os_default.rb`):
   - Stops and disables the default Redis service provided by the OS
   - Resources: service (1) with actions ['stop', 'disable']

7. **redisio::configure** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/configure.rb`):
   - Uses custom resource: redisio_configure['redis-servers']
     - Provider: /workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/providers/configure.rb
     - Configures Redis instances based on node['redisio']['servers']
   - For each Redis server (in this case, one server on port 6379):
     - Creates service based on job_control (systemd, initd, upstart, or rcinit)
     - If systemd: service[redis@6379]
     - If initd/upstart/rcinit: service[redis6379]
   - Resources: redisio_configure (1), service (1)

8. **ruby_block** (`cookbooks/cache/recipes/default.rb`):
   - Executes ruby_block 'fix_redis_config'
   - Modifies /etc/redis/6379.conf to remove specific configuration lines:
     - replica-serve-stale-data
     - replica-read-only
     - repl-ping-replica-period
     - client-output-buffer-limit
     - replica-priority
   - Resources: ruby_block (1)

9. **redisio::enable** (`/workspace/source/migration-dependencies/cookbook_artifacts/redisio-cac70a2ec9102cac4f5391358c8565d244f5d4db/recipes/enable.rb`):
   - For each Redis server (in this case, one server on port 6379):
     - Starts and enables the Redis service
     - If systemd: service[redis@6379]
     - If initd/upstart/rcinit: service[redis6379]
   - Resources: service (1) with actions [:start, :enable]

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- memcached
- redis-server (if using package install)
- build-essential (if building from source)

**Service dependencies**:
- memcached.service
- redis@6379.service (systemd) or redis6379 (initd/upstart/rcinit)

## Checks for the Migration

**Files to verify**:
- /etc/memcached.conf (Debian) or /etc/sysconfig/memcached (RHEL)
- /etc/redis/6379.conf
- /var/log/redis/ (directory)
- /var/lib/redis/ (data directory)
- /etc/systemd/system/redis@.service (if using systemd)
- /etc/init.d/redis6379 (if using initd)
- /etc/init/redis6379.conf (if using upstart)

**Service endpoints to check**:
- Ports listening:
  - 11211 (TCP and UDP) - Memcached
  - 6379 (TCP) - Redis
- Network interfaces: 0.0.0.0 (both services listen on all interfaces)

**Templates rendered**:
- redis.conf.erb → /etc/redis/6379.conf (1 instance)
- redis@.service.erb → /etc/systemd/system/redis@.service (if using systemd)
- redis.init.erb → /etc/init.d/redis6379 (if using initd)
- redis.upstart.conf.erb → /etc/init/redis6379.conf (if using upstart)

## Pre-flight checks:
```bash
# Memcached checks
# Service status
systemctl status memcached
ps aux | grep memcached

# Connection test
echo stats | nc localhost 11211
memcached-tool localhost:11211 stats
memcached-tool localhost:11211 display

# Configuration validation
cat /etc/memcached.conf  # Debian
cat /etc/sysconfig/memcached  # RHEL
grep -E 'memory|port|listen|maxconn' /etc/memcached.conf

# Logs
journalctl -u memcached -f
tail -f /var/log/memcached.log  # if log file is configured

# Network listening
netstat -tulpn | grep 11211
ss -tulpn | grep 11211
lsof -i :11211

# Redis checks
# Service status
systemctl status redis@6379  # if using systemd
service redis6379 status  # if using initd/upstart

# Connection test
redis-cli -h localhost -p 6379 ping
redis-cli -h localhost -p 6379 -a redis_secure_password_123 ping
redis-cli -h localhost -p 6379 -a redis_secure_password_123 info

# Authentication test
redis-cli -h localhost -p 6379 ping  # Should fail without password
redis-cli -h localhost -p 6379 -a redis_secure_password_123 ping  # Should return PONG

# Configuration validation
cat /etc/redis/6379.conf
grep -E 'port|requirepass|databases' /etc/redis/6379.conf

# Verify config modifications
grep -L "replica-serve-stale-data" /etc/redis/6379.conf
grep -L "replica-read-only" /etc/redis/6379.conf
grep -L "repl-ping-replica-period" /etc/redis/6379.conf
grep -L "client-output-buffer-limit" /etc/redis/6379.conf
grep -L "replica-priority" /etc/redis/6379.conf

# Logs
tail -f /var/log/redis/redis-server-6379.log
journalctl -u redis@6379 -f  # if using systemd

# Network listening
netstat -tulpn | grep 6379
ss -tulpn | grep 6379
lsof -i :6379

# Data directories
ls -lah /var/lib/redis/
df -h /var/lib/redis/

# Memory usage
ps aux | grep redis-server | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
```