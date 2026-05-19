---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up a single Redis instance with authentication on port 6379 and includes Memcached with default settings. The cookbook applies a configuration fix to Redis by removing specific replication-related settings.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis 6379**: Primary Redis instance
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password, replication settings removed
  - Log Directory: /var/log/redis

- **Memcached**: Default Memcached instance
  - Location/Path: Default (determined by memcached cookbook)
  - Port/Socket: Default (likely 11211)
  - Key Config: Default settings from memcached cookbook

## File Structure

```
cookbooks/cache/recipes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes memcached recipe to install and configure Memcached
     - Resources: include_recipe (1)
   - Sets Redis configuration attributes:
     - Configures a single Redis instance on port 6379
     - Enables authentication with password 'redis_secure_password_123'
     - Disables replica-serve-stale-data setting
   - Creates Redis log directory at /var/log/redis
     - Resources: directory (1)
   - Includes redisio recipe to install and configure Redis
     - Resources: include_recipe (1)
   - Applies a configuration fix to remove specific replication settings from Redis config
     - Removes lines containing: replica-serve-stale-data, replica-read-only, repl-ping-replica-period, client-output-buffer-limit, replica-priority
     - Resources: ruby_block (1)
   - Includes redisio::enable recipe to enable and start Redis service
     - Resources: include_recipe (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis server packages (installed by redisio cookbook)
- Memcached packages (installed by memcached cookbook)

**Service dependencies**:
- redis service (managed by redisio cookbook)
- memcached service (managed by memcached cookbook)

## Credentials

**Detection Summary**: 1 credential detected in 1 file

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
- Ports listening:
  - Redis: 6379
  - Memcached: 11211 (default)
- Network interfaces: Likely 127.0.0.1 (localhost) by default

**Templates rendered**:
- No templates directly rendered by this cookbook
- Redis configuration is managed by the redisio cookbook
- Memcached configuration is managed by the memcached cookbook

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
grep -v "^#" /etc/redis/6379.conf | grep -v "^$"
grep "requirepass" /etc/redis/6379.conf
grep -E "replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority" /etc/redis/6379.conf
# These settings should be removed by the ruby_block

# Redis logs
ls -la /var/log/redis
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

# Memcached network listening
netstat -tulpn | grep 11211
ss -tlnp | grep memcached
lsof -i :11211

# Memory usage
free -m
ps aux | grep redis | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
ps aux | grep memcached | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
```