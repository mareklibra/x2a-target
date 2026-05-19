---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up Redis with authentication and custom configuration, while relying on the memcached cookbook for Memcached setup. The cookbook manages a single Redis instance running on port 6379.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis 6379**: Main Redis instance
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password 'redis_secure_password_123'

- **Memcached**: Default Memcached instance (configured via dependency)
  - Location/Path: Default Memcached configuration
  - Port/Socket: Default (likely 11211)
  - Key Config: Default configuration from memcached cookbook

## File Structure

```
cookbooks/cache/recipes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes memcached recipe from external dependency
   - Sets Redis configuration attributes:
     - port: 6379
     - requirepass: 'redis_secure_password_123'
     - replicaservestaledata: nil
   - Creates Redis log directory at /var/log/redis
     - Owner: redis
     - Group: redis
     - Mode: 0755
     - Recursive: true
   - Includes redisio recipe from external dependency
   - Executes ruby_block 'fix_redis_config' to modify Redis configuration:
     - Removes lines containing:
       - replica-serve-stale-data
       - replica-read-only
       - repl-ping-replica-period
       - client-output-buffer-limit
       - replica-priority
   - Includes redisio::enable recipe from external dependency
   - Resources: include_recipe (3), directory (1), ruby_block (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis server (installed via redisio cookbook)
- Memcached server (installed via memcached cookbook)

**Service dependencies**:
- redis@6379 (systemd service)
- memcached (systemd service)

## Credentials

**Detection Summary**: 1 credential detected across 1 file

**Source**:
  - **Provider**: Hardcoded
  - **URL**: N/A
  - **Path**: N/A

### Redis Authentication Password

- **Variable(s)**: `node.default['redisio']['servers'][0]['requirepass']`
- **Source file(s)**: cookbooks/cache/recipes/default.rb
- **Current storage**: hardcoded
- **Usage context**: Redis server authentication password used to secure Redis instance

## Checks for the Migration

**Files to verify**:
- /etc/redis/6379.conf
- /var/log/redis/
- /etc/memcached.conf (likely path for memcached configuration)

**Service endpoints to check**:
- Ports listening: 6379 (Redis), 11211 (likely Memcached default)
- Unix sockets: None specified
- Network interfaces: Default (likely 0.0.0.0 or 127.0.0.1)

**Templates rendered**:
- No templates in this cookbook (Redis configuration is managed by the redisio cookbook)

## Pre-flight checks:

```bash
# Redis Service status
systemctl status redis@6379
ps aux | grep redis

# Redis connectivity
redis-cli -h localhost -p 6379 ping
redis-cli -h localhost -p 6379 -a 'redis_secure_password_123' ping
redis-cli -h localhost -p 6379 -a 'redis_secure_password_123' info server

# Redis configuration validation
grep -v '^#' /etc/redis/6379.conf | grep -v '^$'
grep 'requirepass' /etc/redis/6379.conf
grep -E 'replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority' /etc/redis/6379.conf

# Redis log directory
ls -la /var/log/redis/
tail -f /var/log/redis/redis_6379.log

# Memcached Service status
systemctl status memcached
ps aux | grep memcached

# Memcached connectivity
echo stats | nc localhost 11211
memcached-tool localhost:11211 stats

# Network listening
netstat -tulpn | grep 6379
netstat -tulpn | grep 11211
ss -tlnp | grep redis
ss -tlnp | grep memcached
lsof -i :6379
lsof -i :11211

# Memory usage
ps aux | grep redis | grep -v grep
ps aux | grep memcached | grep -v grep
free -m

# Service logs
journalctl -u redis@6379 -n 50
journalctl -u memcached -n 50
```