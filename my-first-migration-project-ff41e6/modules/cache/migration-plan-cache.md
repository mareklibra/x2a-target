---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up Redis with authentication and custom configuration, including a specific port (6379) and password. The cookbook relies on external dependencies for the actual installation and configuration of both services.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis**:
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password 'redis_secure_password_123'

- **Memcached**:
  - Location/Path: Not explicitly defined in this cookbook (handled by dependency)
  - Port/Socket: Not explicitly defined in this cookbook (handled by dependency)
  - Key Config: Not explicitly defined in this cookbook (handled by dependency)

## File Structure

```
cookbooks/cache/recipes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes the memcached recipe from external dependency
   - Sets Redis configuration attributes:
     - Port: 6379
     - Password: 'redis_secure_password_123'
     - Disables replicaservestaledata
   - Creates Redis log directory at /var/log/redis
     - Owner: redis
     - Group: redis
     - Mode: 0755
     - Recursive: true
   - Includes the redisio recipe from external dependency
   - Executes a ruby_block to modify Redis configuration:
     - Removes specific replication and client buffer settings from /etc/redis/6379.conf
     - Removes lines containing:
       - replica-serve-stale-data
       - replica-read-only
       - repl-ping-replica-period
       - client-output-buffer-limit
       - replica-priority
   - Includes the redisio::enable recipe from external dependency
   - Resources: include_recipe (3), directory (1), ruby_block (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis (installed by redisio cookbook)
- Memcached (installed by memcached cookbook)

**Service dependencies**:
- redis (managed by redisio cookbook)
- memcached (managed by memcached cookbook)

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
- **Usage context**: Redis server authentication password used in Redis configuration

## Checks for the Migration

**Files to verify**:
- /etc/redis/6379.conf
- /var/log/redis (directory)

**Service endpoints to check**:
- Ports listening: 6379 (Redis)
- Unix sockets: None explicitly defined
- Network interfaces: Not explicitly defined, likely default to all interfaces

**Templates rendered**:
- No templates are directly rendered by this cookbook

## Pre-flight checks:
```bash
# Redis Service status
systemctl status redis-server@6379
systemctl status redis
ps aux | grep redis

# Redis connectivity
redis-cli -p 6379 ping
redis-cli -p 6379 -a 'redis_secure_password_123' ping
redis-cli -p 6379 -a 'redis_secure_password_123' info server

# Redis configuration validation
grep -v "^#" /etc/redis/6379.conf | grep -v "^$"
grep "requirepass" /etc/redis/6379.conf
grep -E "replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority" /etc/redis/6379.conf

# Redis log directory
ls -la /var/log/redis
stat -c "%U %G %a" /var/log/redis

# Redis logs
tail -f /var/log/redis/*.log

# Network listening
netstat -tulpn | grep 6379
ss -tlnp | grep redis
lsof -i :6379

# Memcached Service status
systemctl status memcached
ps aux | grep memcached

# Memcached connectivity
echo stats | nc localhost 11211
memcached-tool localhost:11211 stats

# Network listening for Memcached
netstat -tulpn | grep memcached
ss -tlnp | grep memcached
lsof -i :11211
```