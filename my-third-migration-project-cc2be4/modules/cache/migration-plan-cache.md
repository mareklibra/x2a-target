---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook is intended to configure Memcached and Redis caching services. The supplied execution tree contains only `cookbooks/cache/recipes/default.rb`; the referenced `memcached`, `redisio`, and `redisio::enable` recipes are unavailable. The visible recipe defines one Redis instance on port `6379`, creates `/var/log/redis`, removes selected replication-related directives from `/etc/redis/6379.conf`, and attempts to enable Redis. Resource properties, package names, service names, and the reported Redis password require verification against the underlying recipe source before implementation.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis-6379**: Redis cache instance configured through `redisio`
  - Location/Path: `/etc/redis/6379.conf`
  - Port/Socket: TCP `6379`
  - Key Config: The recipe assigns port `6379`, sets `replicaservestaledata` to `nil`, and reportedly assigns a `requirepass` value. The password finding requires source verification.
  - Log Directory: `/var/log/redis`; ownership, group, mode, and recursive-creation properties require verification.
  - Post-processing: The recipe reportedly removes the following directives when present:
    - `replica-serve-stale-data`
    - `replica-read-only`
    - `repl-ping-replica-period`
    - `client-output-buffer-limit`
    - `replica-priority`
  - Enablement: Attempts to include `redisio::enable`; the referenced recipe is unavailable, so the Redis service name and enablement behavior are unknown.

- **Memcached**: Referenced cache service
  - Location/Path: Unknown; the `memcached` recipe is unavailable.
  - Port/Socket: Unknown.
  - Key Config: Unknown.
  - Package, service name, configuration path, bind address, log path, and authentication settings must be obtained from the missing cookbook before migration.

## File Structure

**MANDATORY: Preserve this section from the original plan.**

**Recipes:**
```text
cookbooks/cache/recipes/default.rb
```

**Providers:**
```text
```

No provider files were shown.

**Templates:**
```text
```

No templates were shown.

**Attributes:**
```text
```

No attribute files were shown. The Redis configuration is assigned directly in `cookbooks/cache/recipes/default.rb`.

**Files:**
```text
```

No static files were shown.

The following referenced recipe files are intentionally not listed because they were explicitly reported as unavailable:

- `memcached`
- `redisio`
- `redisio::enable`

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes `memcached` through `include_recipe 'memcached'`.
   - The `memcached` recipe is unavailable. Its package installation, service management, port configuration, and templates cannot be determined from the supplied execution tree.
   - Assigns one Redis server configuration to `node.default['redisio']['servers']`:
     - Port: `6379`
     - Password: a hardcoded value is reported in the recipe but requires source verification.
     - `replicaservestaledata`: `nil`
   - Creates `/var/log/redis` through `directory[/var/log/redis]`.
   - The resource is confirmed, but its owner, group, mode, and recursive settings require verification from the detailed recipe source. The directory references the `redis` owner and group only if those properties are confirmed in the source.
   - Includes `redisio` through `include_recipe 'redisio'`.
   - The `redisio` recipe is unavailable. Redis package names, service names, configuration templates, users, and directories cannot be determined from the supplied execution tree.
   - Runs `ruby_block[fix_redis_config]`.
   - The resource is confirmed, but its file path, existence check, and directive-removal behavior require verification from the detailed recipe source. The reported intended target is `/etc/redis/6379.conf`, with removal of:
     - `replica-serve-stale-data`
     - `replica-read-only`
     - `repl-ping-replica-period`
     - `client-output-buffer-limit`
     - `replica-priority`
   - If the reported behavior is confirmed, implement it with a conditional Ansible `replace`, `lineinfile`, or equivalent task that removes complete matching lines only when `/etc/redis/6379.conf` exists.
   - Includes `redisio::enable` through `include_recipe 'redisio::enable'`.
   - The `redisio::enable` recipe is unavailable. Confirm the Redis service name before implementing service enablement or restart behavior.

**Visible resource counts:**

- `directory`: 1
- `ruby_block`: 1
- `include_recipe`: 3
  - `memcached`
  - `redisio`
  - `redisio::enable`

No `.each` loops were identified in the supplied recipe.

## Dependencies

**External cookbook dependencies**: `memcached` version constraint `~> 6.0` and `redisio` with no version constraint are reported from cookbook metadata, but metadata analysis was not supplied. Verify these values from `metadata.rb`.

**System package dependencies**: None confirmed. The visible recipe contains no `package` resources. Package names must be obtained from the unavailable `memcached` and `redisio` recipes before adding Ansible package tasks.

**Service dependencies**:
- Memcached service: unknown because `memcached` is unavailable.
- Redis service: delegated to unavailable `redisio::enable`.
- Redis service name and restart behavior: unknown.
- Supported platforms reported by the source are Ubuntu `>= 18.04` and CentOS `>= 7.0`; verify these values from `metadata.rb`.

## Credentials

**Detection Summary**: 1 reported credential in 1 file; the finding and value require verification against the recipe source.

**Source**:
  - **Provider**: Hardcoded in Chef recipe
  - **URL**: None detected
  - **Path**: None detected

### Redis authentication password

- **Variable(s)**: `node.default['redisio']['servers'][0]['requirepass']`
- **Source file(s)**: `cookbooks/cache/recipes/default.rb`
- **Current storage**: Reported hardcoded plaintext value; source verification required
- **Reported value**: `redis_secure_password_123`
- **Usage context**: Reportedly passed as the Redis `requirepass` setting for `Redis-6379`.
- **Migration handling**:
  - Store the confirmed secret in an Ansible Vault-encrypted variable or AAP credential.
  - Do not place the plaintext value in inventory, committed variable files, templates, task output, or migration documentation after verification.
  - Use `no_log: true` on tasks that handle the secret where appropriate.
  - Apply restrictive permissions to any generated Redis configuration containing the password.

No `data_bag_item`, `encrypted_data_bag_item`, `chef_vault_item`, `ChefVault::Item`, `conjur_variable`, CyberArk data bag, or environment-variable secret reference was identified in the supplied files.

## Checks for the Migration

**Files to verify**:
- `cookbooks/cache/recipes/default.rb`
- `/etc/redis/6379.conf`
- `/var/log/redis`

The following outputs require the unavailable recipes before they can be confirmed:

- Redis executable and package paths
- Redis systemd unit
- Memcached configuration
- Memcached log files
- Memcached systemd unit

**Service endpoints to check**:
- `Redis-6379`: TCP port `6379`
- `Memcached`: port or Unix socket unknown until the `memcached` recipe is obtained

**Templates rendered**:
- None identified
- Render count: `0` confirmed templates
- No `.erb` files were shown or referenced in the supplied execution tree

## Pre-flight checks:

```bash
# Confirm the missing cookbook implementations before migration.
test -f cookbooks/cache/recipes/default.rb
test -f cookbooks/memcached/recipes/default.rb
test -f cookbooks/redisio/recipes/default.rb
test -f cookbooks/redisio/recipes/enable.rb

# Redis-6379: replace <redis-service-name> only after confirming it from redisio.
systemctl status <redis-service-name>
systemctl is-enabled <redis-service-name>
ps aux | grep '[r]edis'

# Redis-6379: verify the configured TCP endpoint.
ss -tlnp | grep ':6379'
nc -zv 127.0.0.1 6379

# Redis-6379: use the migrated secret without placing it in shell history.
redis-cli -p 6379 -a "$REDIS_PASSWORD" PING
redis-cli -p 6379 PING

# Redis-6379: verify the configuration file without printing the password.
test -f /etc/redis/6379.conf
grep -nE '^(port|replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority)' /etc/redis/6379.conf
grep -nE '^(replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority)' /etc/redis/6379.conf

# Redis-6379: verify the log directory after confirming its expected properties.
test -d /var/log/redis
stat -c '%A %U %G %n' /var/log/redis
ls -ld /var/log/redis

# Redis-6379: inspect available logs without assuming an unconfirmed filename.
find /var/log/redis -maxdepth 1 -type f -print

# Redis-6379: validate service health after configuration changes.
systemctl is-active <redis-service-name>
redis-cli -p 6379 -a "$REDIS_PASSWORD" PING

# Memcached: these checks are blocked until the missing recipe provides
# the package, service name, port or socket, configuration path, and bind address.
echo "Memcached checks pending recovery of the memcached cookbook implementation."
```