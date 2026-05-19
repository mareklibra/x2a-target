---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up Redis with authentication and custom configuration, including a specific port (6379) and password. The cookbook also includes a workaround to fix Redis configuration by removing certain replication-related settings.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis**:
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password, certain replication settings removed
  - Log Directory: /var/log/redis

- **Memcached**:
  - Default configuration (no specific customization visible in the cookbook)

## File Structure

```
cookbooks/cache/recipes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes the memcached recipe (from external dependency)
     - Resources: include_recipe (1)
   - Sets Redis configuration attributes:
     - Port: 6379
     - Password: redis_secure_password_123
     - Disables replicaservestaledata
   - Creates Redis log directory at /var/log/redis
     - Resources: directory (1)
   - Includes the redisio recipe (from external dependency)
     - Resources: include_recipe (1)
   - Executes a ruby_block to fix Redis configuration
     - Removes specific replication-related settings from /etc/redis/6379.conf:
       - replica-serve-stale-data
       - replica-read-only
       - repl-ping-replica-period
       - client-output-buffer-limit
       - replica-priority
     - Resources: ruby_block (1)
   - Includes the redisio::enable recipe (from external dependency)
     - Resources: include_recipe (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis server
- Memcached server

**Service dependencies**:
- redis (likely managed by redisio cookbook)
- memcached (likely managed by memcached cookbook)

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
- Memcached configuration files (location depends on OS)

**Service endpoints to check**:
- Ports listening: 6379 (Redis)
- Ports listening: 11211 (default Memcached port, assuming default configuration)

**Templates rendered**:
- No templates directly rendered by this cookbook

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

# Redis configuration verification
grep requirepass /etc/redis/6379.conf
grep -v "replica-serve-stale-data\|replica-read-only\|repl-ping-replica-period\|client-output-buffer-limit\|replica-priority" /etc/redis/6379.conf

# Redis log directory
ls -la /var/log/redis

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
free -m
ps aux | grep redis | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
ps aux | grep memcached | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS

# Logs
tail -f /var/log/redis/redis_6379.log
journalctl -u redis-server -f
journalctl -u redis@6379 -f
journalctl -u memcached -f
```