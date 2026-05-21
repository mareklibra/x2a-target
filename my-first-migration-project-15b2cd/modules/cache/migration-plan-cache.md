---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up Redis with authentication and performs some configuration file modifications. The cookbook relies heavily on external cookbooks for the actual installation and configuration of both services.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis**:
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password, certain replication settings disabled via ruby_block

- **Memcached**:
  - Location/Path: Not specified in this cookbook (handled by external memcached cookbook)
  - Port/Socket: Not specified in this cookbook (typically 11211)
  - Key Config: Not specified in this cookbook

## File Structure

```
cookbooks/cache/recipes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes the memcached recipe from external cookbook
     - Resources: include_recipe (1)
   - Sets Redis configuration attributes:
     - Configures Redis to run on port 6379
     - Enables authentication with password 'redis_secure_password_123'
     - Disables replica-serve-stale-data setting
   - Creates Redis log directory at /var/log/redis
     - Resources: directory (1)
   - Includes the redisio recipe from external cookbook
     - Resources: include_recipe (1)
   - Executes a ruby_block to modify Redis configuration file
     - Removes several replication-related settings from /etc/redis/6379.conf
     - Resources: ruby_block (1)
   - Includes the redisio::enable recipe from external cookbook
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
- **Usage context**: Redis server authentication password used to secure Redis instance

## Checks for the Migration

**Files to verify**:
- /etc/redis/6379.conf
- /var/log/redis (directory)
- Memcached configuration files (location depends on memcached cookbook)

**Service endpoints to check**:
- Ports listening: 6379 (Redis)
- Memcached port (typically 11211, but not specified in this cookbook)

**Templates rendered**:
- No templates are directly rendered by this cookbook

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

# Redis logs
tail -f /var/log/redis/redis_6379.log

# Redis network listening
netstat -tulpn | grep 6379
ss -tlnp | grep redis
lsof -i :6379

# Redis data directories
ls -lah /var/lib/redis/
ls -lah /var/log/redis/

# Memcached Service status
systemctl status memcached
ps aux | grep memcached

# Memcached connectivity
echo "stats" | nc localhost 11211
memcached-tool localhost:11211 stats

# Memcached configuration validation
cat /etc/memcached.conf

# Memcached network listening
netstat -tulpn | grep memcached
ss -tlnp | grep memcached
lsof -i :11211

# Memcached logs
journalctl -u memcached -f
```