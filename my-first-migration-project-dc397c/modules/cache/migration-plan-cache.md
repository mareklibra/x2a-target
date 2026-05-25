---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up a single Redis instance with authentication on port 6379 and includes Memcached with default settings. The cookbook applies a configuration fix to Redis by removing specific replication-related settings.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis**:
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password 'redis_secure_password_123'
  - Log Directory: /var/log/redis

- **Memcached**:
  - Location/Path: Default (determined by memcached cookbook)
  - Port/Socket: Default (typically 11211)
  - Key Config: Default settings from memcached cookbook

## File Structure

```
cookbooks/cache/recipes/default.rb
cookbooks/cache/metadata.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **memcached** (dependency cookbook):
   - Includes the memcached recipe from an external cookbook
   - Resources: include_recipe (1)

2. **redis configuration** (`cookbooks/cache/recipes/default.rb`):
   - Sets Redis server attributes with port 6379 and password authentication
   - Creates Redis log directory at /var/log/redis
   - Resources: directory (1)

3. **redisio** (dependency cookbook):
   - Includes the redisio recipe from an external cookbook
   - Resources: include_recipe (1)

4. **redis config fix** (`cookbooks/cache/recipes/default.rb`):
   - Applies a configuration fix to remove specific replication-related settings
   - Modifies /etc/redis/6379.conf to remove:
     - replica-serve-stale-data
     - replica-read-only
     - repl-ping-replica-period
     - client-output-buffer-limit
     - replica-priority
   - Resources: ruby_block (1)

5. **redisio::enable** (dependency cookbook):
   - Includes the redisio::enable recipe to enable and start Redis service
   - Resources: include_recipe (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis server package (installed by redisio cookbook)
- Memcached package (installed by memcached cookbook)

**Service dependencies**:
- redis service (managed by redisio cookbook)
- memcached service (managed by memcached cookbook)

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
- /var/log/redis (directory)
- Memcached configuration file (location depends on memcached cookbook)

**Service endpoints to check**:
- Ports listening: 6379 (Redis), 11211 (Memcached default)
- Unix sockets: None specified
- Network interfaces: Default (0.0.0.0)

**Templates rendered**:
- None directly in this cookbook (Redis config is managed by redisio cookbook)

## Pre-flight checks:

```bash
# Redis Service status
systemctl status redis-server
systemctl status redis@6379
ps aux | grep redis

# Redis connectivity
redis-cli -p 6379 ping
redis-cli -p 6379 -a 'redis_secure_password_123' ping
redis-cli -p 6379 -a 'redis_secure_password_123' info server

# Redis configuration validation
cat /etc/redis/6379.conf | grep -E 'port|requirepass'
cat /etc/redis/6379.conf | grep -E 'replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority'

# Redis logs
ls -la /var/log/redis/
tail -f /var/log/redis/redis_6379.log

# Redis network listening
netstat -tulpn | grep 6379
ss -tlnp | grep redis
lsof -i :6379

# Memcached Service status
systemctl status memcached
ps aux | grep memcached

# Memcached connectivity
echo stats | nc localhost 11211
memcached-tool localhost:11211 stats

# Memcached configuration validation
cat /etc/memcached.conf

# Memcached logs
journalctl -u memcached -f

# Memcached network listening
netstat -tulpn | grep 11211
ss -tlnp | grep memcached
lsof -i :11211

# Directory permissions
ls -la /var/log/redis/
```