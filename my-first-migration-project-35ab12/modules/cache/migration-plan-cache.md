---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures caching services through the external `memcached` and `redisio` cookbooks. The analyzed cookbook explicitly configures one Redis instance on port `6379`, creates `/var/log/redis`, removes selected replication directives from `/etc/redis/6379.conf`, and enables Redis. The `memcached`, `redisio`, and `redisio::enable` recipes are referenced but unavailable in the execution tree, so their package names, service names, configuration paths, and ports must be obtained from the external dependencies. The Redis settings and hardcoded password are reported by the recipe content but require source verification against the available analysis.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis server on port 6379**: Redis cache instance configured through `node['redisio']['servers']`
  - Location/Path: `/etc/redis/6379.conf`
  - Port/Socket: TCP `6379`
  - Key Config: Password authentication is reported as `redis_secure_password_123`; `replicaservestaledata` is reported as `nil`. These inline recipe values require source verification.
  - Log Directory: `/var/log/redis`
  - Log Directory Ownership: Reported as `redis:redis`, mode `0755`, with recursive creation; these resource properties require verification because they are not present in the supplied structured analysis.
  - Post-configuration cleanup: Removes `replica-serve-stale-data`, `replica-read-only`, `repl-ping-replica-period`, `client-output-buffer-limit`, and `replica-priority` from the configuration file when present.

- **Memcached**: External cache service referenced by `include_recipe 'memcached'`
  - Location/Path: Not specified by the analyzed cookbook
  - Port/Socket: Not specified by the analyzed cookbook
  - Key Config: Package, service name, configuration path, and port are delegated to the unavailable external recipe.
  - Dependency Status: `memcached` recipe was not found in the execution tree.

Do not infer Memcached package, configuration, service, or port behavior from this cookbook alone.

## File Structure

**MANDATORY: Preserve this section from the original plan.**

**Recipes:**
```text
cookbooks/cache/recipes/default.rb
```

**Providers:**
```text
```

No provider files are present in the execution tree.

**Templates:**
```text
```

No templates are present in the execution tree.

**Attributes:**
```text
```

No separate attribute files are present. Reported Redis settings are assigned inline in `cookbooks/cache/recipes/default.rb`.

**Files:**
```text
```

No static files deployed by `cookbook_file` or `remote_file` are present in the execution tree.

## Module Explanation

The cookbook performs operations in this order:

1. **`cache::default`** (`cookbooks/cache/recipes/default.rb`)
   - Includes the external `memcached` recipe.
   - Configures the Redis server collection with one reported Redis entry.
   - Creates `/var/log/redis`.
   - Includes the external `redisio` recipe.
   - Removes selected replication-related directives from `/etc/redis/6379.conf`.
   - Includes the external `redisio::enable` recipe.
   - No `.each` loops are identified in the supplied execution tree.

2. **`memcached`** (external recipe; unavailable)
   - Referenced by `include_recipe 'memcached'`.
   - The recipe was not found in the execution tree.
   - Its package, service, configuration path, port, and resources cannot be determined from this cookbook.

3. **`directory[/var/log/redis]`** (`cookbooks/cache/recipes/default.rb`)
   - Creates `/var/log/redis`.
   - The recipe content reportedly specifies owner `redis`, group `redis`, mode `0755`, and recursive creation.
   - These properties are not present in the supplied structured analysis and must be verified against the recipe source.
   - No separate `group[redis]` resource is identified.

4. **`redisio`** (external recipe; unavailable)
   - Referenced by `include_recipe 'redisio'`.
   - The recipe was not found in the execution tree.
   - It is expected to install or configure Redis and create `/etc/redis/6379.conf`, but exact package and service behavior cannot be confirmed from this cookbook.

5. **`ruby_block[fix_redis_config]`** (`cookbooks/cache/recipes/default.rb`)
   - Checks whether `/etc/redis/6379.conf` exists.
   - Reads the file when present.
   - Removes complete lines matching:
     - `^replica-serve-stale-data.*$`
     - `^replica-read-only.*$`
     - `^repl-ping-replica-period.*$`
     - `^client-output-buffer-limit.*$`
     - `^replica-priority.*$`
   - Writes the modified content back to `/etc/redis/6379.conf`.
   - Does nothing when the file does not exist.
   - The Ansible equivalent should use conditional `ansible.builtin.replace` tasks or a controlled configuration template.

6. **`redisio::enable`** (external recipe; unavailable)
   - Referenced by `include_recipe 'redisio::enable'`.
   - The recipe was not found in the execution tree.
   - The migration must enable and start Redis using the service name supplied by the external Redis role or target platform.
   - Redis enablement must occur after configuration cleanup.

## Dependencies

**External cookbook dependencies**:
- `memcached`, version constraint `~> 6.0`
- `redisio`, with no version constraint shown

**Supported platforms**:
- Ubuntu `>= 18.04`
- CentOS `>= 7.0`

**System package dependencies**:
- No package resource is visible in the analyzed cookbook.
- Exact package names are not verified and must be obtained from the external `memcached` and `redisio` implementations.
- Do not assign package names to this cookbook without dependency-source verification.

**Service dependencies**:
- Memcached service behavior is delegated to the unavailable `memcached` recipe.
- Redis installation and service behavior are delegated to the unavailable `redisio` and `redisio::enable` recipes.
- `/etc/redis/6379.conf` must exist before `ruby_block[fix_redis_config]` or its Ansible equivalent runs.
- Redis must be enabled and started after configuration deployment and cleanup.

## Credentials

**Detection Summary**: 1 reported credential in 1 file; the credential finding requires verification against the recipe source because credential-analysis output was not supplied.

**Source**:
  - **Provider**: Hardcoded in the Chef recipe, as reported
  - **URL**: None detected
  - **Path**: None detected

### Redis authentication password

- **Variable**: `node['redisio']['servers'][0]['requirepass']`
- **Literal value in source**: `redis_secure_password_123`, requiring source verification
- **Source file(s)**: `cookbooks/cache/recipes/default.rb`
- **Current storage**: Reported as hardcoded inline in a Chef recipe
- **Usage context**: Redis `requirepass` authentication for the Redis server listening on port `6379`
- **Migration storage**: Store the password in Ansible Vault, preferably as `redis_requirepass` or `redis_servers[0].requirepass`
- **Required handling**: Do not place the plaintext password in committed Ansible tasks, defaults files, templates, or generated configuration under source control.

No Chef data bags, encrypted data bags, Chef Vault calls, CyberArk integration, Conjur calls, HashiCorp Vault references, AWS Secrets Manager references, environment-variable secrets, TLS certificate references, or database connection strings were detected.

## Checks for the Migration

**Files to verify**:
- `cookbooks/cache/recipes/default.rb`
- `/etc/redis/6379.conf`
- `/var/log/redis`

**Service endpoints to check**:
- Redis TCP port `6379`
- Memcached endpoint: Not specified by the analyzed cookbook

**Unix sockets**:
- None specified by the analyzed cookbook.

**Templates rendered**:
- None; render count: `0`

**Configuration directives to verify as absent from Redis configuration**:
- `replica-serve-stale-data`
- `replica-read-only`
- `repl-ping-replica-period`
- `client-output-buffer-limit`
- `replica-priority`

## Pre-flight checks

### Redis instance on port 6379

```bash
# Identify the service name supplied by the external redisio implementation.
systemctl status <redis-service-name>
systemctl is-enabled <redis-service-name>
systemctl is-active <redis-service-name>

# Verify the Redis listener.
ss -ltnp | grep ':6379'
netstat -ltnp 2>/dev/null | grep ':6379'

# Verify the configuration file and core settings.
test -f /etc/redis/6379.conf
grep -nE '^(port|requirepass)' /etc/redis/6379.conf

# Verify that all five directives were removed.
! grep -nE '^(replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority)' /etc/redis/6379.conf

# Verify the log directory.
test -d /var/log/redis
stat -c '%U %G %a %n' /var/log/redis

# Authenticate using the migrated vaulted password.
redis-cli -h 127.0.0.1 -p 6379 -a '<redis-password>' PING

# Functional write/read/delete test.
redis-cli -h 127.0.0.1 -p 6379 -a '<redis-password>' SET chef_migration_check success
redis-cli -h 127.0.0.1 -p 6379 -a '<redis-password>' GET chef_migration_check
redis-cli -h 127.0.0.1 -p 6379 -a '<redis-password>' DEL chef_migration_check

# Review service logs.
journalctl -u <redis-service-name> --no-pager -n 100
```

Expected results:

- The Redis service is enabled and active.
- Redis listens on TCP port `6379`.
- `PING` returns `PONG`.
- The functional `GET` returns `success`.
- `/var/log/redis` exists with the verified ownership and permissions.
- No targeted replication directives remain in `/etc/redis/6379.conf`.
- The service journal contains no configuration, authentication, or startup errors.

### Memcached instance

The `memcached` recipe is unavailable, and this cookbook does not specify a package, service name, configuration path, socket, or port.

After obtaining the external dependency implementation:

```bash
# Replace placeholders with values verified from the memcached dependency.
systemctl status <memcached-service-name>
systemctl is-enabled <memcached-service-name>
systemctl is-active <memcached-service-name>
ss -ltnp | grep ':<verified-memcached-port>'
```

Do not create a definitive Memcached endpoint or health check until the dependency source supplies the exact service name, port, and health procedure.