---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up Redis with authentication on port 6379 and includes a workaround to fix Redis configuration. The cookbook relies on external dependencies for the actual installation and configuration of both services.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis**:
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password, certain replication settings removed via ruby_block

- **Memcached**:
  - Location/Path: Not explicitly defined in this cookbook (handled by dependency)
  - Port/Socket: Not explicitly defined in this cookbook (handled by dependency)
  - Key Config: Uses default settings from the memcached cookbook

## File Structure

```
cookbooks/cache/recipes/default.rb
cookbooks/cache/metadata.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes the memcached recipe from the external memcached cookbook
     - Resources: include_recipe (1)
   - Sets Redis configuration attributes:
     - Configures Redis server on port 6379 with password authentication
     - Disables replicaservestaledata setting
   - Creates Redis log directory at /var/log/redis
     - Resources: directory (1)
   - Includes the redisio recipe from the external redisio cookbook
     - Resources: include_recipe (1)
   - Executes a ruby_block to modify the Redis configuration file
     - Removes several replication-related settings from /etc/redis/6379.conf
     - Resources: ruby_block (1)
   - Includes the redisio::enable recipe to enable and start Redis service
     - Resources: include_recipe (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis server (installed by redisio cookbook)
- Memcached server (installed by memcached cookbook)

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
- **Usage context**: Used for Redis authentication, set in the Redis configuration file

## Checks for the Migration

**Files to verify**:
- /etc/redis/6379.conf
- /var/log/redis (directory)
- Memcached configuration file (location depends on memcached cookbook implementation)

**Service endpoints to check**:
- Ports listening: 6379 (Redis)
- Memcached port (typically 11211, but not explicitly defined in this cookbook)

**Templates rendered**:
- No templates are directly rendered by this cookbook. The Redis configuration is modified after being created by the redisio cookbook.

## Pre-flight checks:

```bash
# Redis Service status
systemctl status redis-server
systemctl status redis@6379
ps aux | grep redis

# Redis connectivity
redis-cli -h localhost -p 6379 ping
redis-cli -h localhost -p 6379 -a 'redis_secure_password_123' ping
redis-cli -h localhost -p 6379 -a 'redis_secure_password_123' info server

# Redis configuration validation
cat /etc/redis/6379.conf | grep -E 'port|requirepass'
cat /etc/redis/6379.conf | grep -E 'replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority'
# These settings should be removed by the ruby_block

# Redis log directory
ls -la /var/log/redis

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

# Memcached network listening
netstat -tulpn | grep memcached
ss -tlnp | grep memcached
lsof -i :11211

# Memory usage
free -m
ps aux | grep redis | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
ps aux | grep memcached | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
```