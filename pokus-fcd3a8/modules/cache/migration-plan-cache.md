---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: The `cache` cookbook configures Memcached through the external `memcached` cookbook and one Redis instance through the external `redisio` cookbook. The locally available recipe creates `/var/log/redis`, configures Redis on port `6379` with a hardcoded password, removes selected replication directives from `/etc/redis/6379.conf`, and includes Redis enablement. The external dependency recipes are unavailable, so their package names, service names, templates, Memcached port, and exact startup behavior must be confirmed before migration.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis server on port 6379**: Redis cache and authenticated service
  - Location/Path: `/etc/redis/6379.conf`
  - Port/Socket: TCP `6379`
  - Key Config:
    - Password: `redis_secure_password_123` in the source recipe; migrate to Ansible Vault or an approved secret store.
    - Log directory: `/var/log/redis`
    - Log directory owner/group: `redis:redis`
    - Log directory mode: `0755`
    - Removed directives:
      - `replica-serve-stale-data`
      - `replica-read-only`
      - `repl-ping-replica-period`
      - `client-output-buffer-limit`
      - `replica-priority`
    - `replicaservestaledata` is explicitly set to `nil`.

- **Memcached**: Memcached service included through the external `memcached` cookbook
  - Location/Path: Not available in the supplied execution tree
  - Port/Socket: Not specified
  - Key Config:
    - Package name, service name, configuration path, bind address, and port are unavailable.
    - Confirm these values from the actual `memcached` cookbook version or selected Ansible implementation.
    - Do not assume a Memcached port or service name.

## File Structure

Only files present in the supplied directory listing and execution tree are included.

**Recipes:**
```text
cookbooks/cache/recipes/default.rb
```

**Providers:**
```text
```

No provider files are shown in the execution tree.

**Templates:**
```text
```

No templates are shown as rendered.

**Attributes:**
```text
```

No attribute files are shown. Redis configuration is assigned directly in `cookbooks/cache/recipes/default.rb`.

**Files:**
```text
```

No static files are shown as deployed.

The following paths are referenced by the recipe but are not cookbook files:

```text
/etc/redis/6379.conf
/var/log/redis
```

## Module Explanation

The cookbook performs operations in this order:

1. **`default`** (`cookbooks/cache/recipes/default.rb`):
   - Includes the external `memcached` recipe.
   - The external recipe is unavailable in the supplied execution tree.
   - Memcached package names, service resource, configuration path, listening port, and bind address cannot be determined from the local cookbook.
   - Represent Memcached as an explicit dependency role or task file only after the external implementation has been inspected.

2. **`default`** (`cookbooks/cache/recipes/default.rb`):
   - Defines one Redis server in `node.default['redisio']['servers']`.
   - Configures:
     - `port`: `'6379'`
     - `requirepass`: `'redis_secure_password_123'`
     - `replicaservestaledata`: `nil`
   - Migrate this as the explicitly named Redis instance listening on TCP port `6379`.

3. **`default`** (`cookbooks/cache/recipes/default.rb`):
   - Creates `/var/log/redis`.
   - Uses owner `redis`, group `redis`, mode `0755`, and recursive creation.
   - Ansible equivalent:
     - `ansible.builtin.file`
     - `path: /var/log/redis`
     - `state: directory`
     - `owner: redis`
     - `group: redis`
     - `mode: '0755'`

4. **`default`** (`cookbooks/cache/recipes/default.rb`):
   - Includes the external `redisio` recipe.
   - The external recipe is unavailable in the supplied execution tree.
   - Its package installation, configuration template, service name, and startup behavior cannot be determined locally.
   - Ensure the Redis instance on port `6379` is configured before applying the post-configuration cleanup.

5. **`default`** (`cookbooks/cache/recipes/default.rb`):
   - Executes `ruby_block[fix_redis_config]`.
   - Reads `/etc/redis/6379.conf` only when the file exists.
   - Removes complete lines matching:
     - `^replica-serve-stale-data.*$`
     - `^replica-read-only.*$`
     - `^repl-ping-replica-period.*$`
     - `^client-output-buffer-limit.*$`
     - `^replica-priority.*$`
   - Writes the modified content back to `/etc/redis/6379.conf`.
   - Use `ansible.builtin.replace` or an equivalent line-removal task.
   - Preserve the behavior that a missing configuration file does not cause a failure.
   - The local recipe does not explicitly restart or reload Redis after this change.
   - Do not add an automatic restart unless required by the selected Redis implementation or operational requirements.

6. **`default`** (`cookbooks/cache/recipes/default.rb`):
   - Includes the external `redisio::enable` recipe.
   - The external recipe is unavailable in the supplied execution tree.
   - Its exact service resource and enablement behavior cannot be determined.
   - Enable and start the Redis service only after confirming the service name generated by the selected Redis package or replacement role.

**Resources from the local recipe, in execution order:**

- `directory[/var/log/redis]`
- `ruby_block[fix_redis_config]`

**External recipe inclusions, in execution order:**

1. `memcached`
2. `redisio`
3. `redisio::enable`

No `.each` iteration loop is shown. The Redis server list contains one configured item: the Redis server on port `6379`.

## Dependencies

**External cookbook dependencies**:

- `memcached`, version constraint `~> 6.0`
- `redisio`, with no version constraint shown

**System package dependencies**:

- No package resources are present in the supplied execution tree.
- Package names supplied by the external `memcached` and `redisio` cookbooks are unavailable and must be confirmed from those implementations.
- No package name is asserted without verification.

**Service dependencies**:

- **Memcached**: Included externally; exact service name unavailable.
- **Redis server on port 6379**: Enabled through `redisio::enable`; exact service name unavailable.

**Platform support from metadata**:

- Ubuntu `>= 18.04`
- CentOS `>= 7.0`

## Credentials

**Detection Summary**: 1 credential detected in 1 file.

**Source**:

- **Provider**: Hardcoded in Chef recipe
- **URL**: None detected
- **Path**: None detected

### Redis Authentication Password

- **Variable**: `node['redisio']['servers'][0]['requirepass']`
- **Literal value in source**: `redis_secure_password_123`
- **Source file(s)**: `cookbooks/cache/recipes/default.rb`
- **Current storage**: Hardcoded plaintext Chef attribute assignment
- **Usage context**: Redis authentication for the Redis server listening on port `6379`
- **Migration storage**: Ansible Vault, an AAP credential, or another approved secret-management system
- **Migration handling**: Pass the vaulted value into the Redis configuration template or role variable and use `no_log: true` where appropriate.

No `data_bag_item`, `encrypted_data_bag_item`, `chef_vault_item`, `ChefVault::Item`, `conjur_variable`, CyberArk data bag, environment-variable secret, TLS certificate, private key, or database connection string was detected.

## Checks for the Migration

**Files to verify**:

- `cookbooks/cache/recipes/default.rb`
- `/etc/redis/6379.conf`
- `/var/log/redis`

**Service endpoints to check**:

- **Redis server on port 6379**: TCP `6379`
- **Memcached**: Port unavailable; confirm from the selected dependency implementation
- Unix sockets: None specified
- Network bind interfaces: Not specified

**Configuration directives to verify**:

The Redis configuration must not contain active lines beginning with:

```text
replica-serve-stale-data
replica-read-only
repl-ping-replica-period
client-output-buffer-limit
replica-priority
```

The Redis configuration must contain authentication configured through the migrated secret value.

**Templates rendered**:

- None shown in the execution tree
- Render count: `0` confirmed local templates

## Pre-flight checks:

```bash
# Verify the Redis log directory for the Redis server on port 6379
ls -ld /var/log/redis
stat -c '%U %G %a %n' /var/log/redis
```

```bash
# Discover the Redis service unit for the Redis server on port 6379
systemctl list-unit-files | grep -Ei 'redis|redis-server'
systemctl list-units --type=service --all | grep -Ei 'redis|redis-server'
```

```bash
# Verify the Redis configuration for the Redis server on port 6379
test -f /etc/redis/6379.conf
grep -nE '^(port|requirepass|replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority)' \
  /etc/redis/6379.conf
```

```bash
# Confirm removed directives are absent from the Redis configuration
if grep -nE '^(replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority)' \
  /etc/redis/6379.conf; then
  echo "Unexpected replication directives remain"
  exit 1
else
  echo "Replication directives were removed"
fi
```

```bash
# Verify the Redis listener on the exact configured port
ss -ltnp | grep ':6379'
```

```bash
# Verify authenticated connectivity to the Redis server on port 6379
REDISCLI_AUTH='REDACTED_REDIS_PASSWORD' redis-cli -p 6379 PING
```

Expected result:

```text
PONG
```

```bash
# Verify the Redis server port and version
REDISCLI_AUTH='REDACTED_REDIS_PASSWORD' \
  redis-cli -p 6379 INFO server | grep -E 'redis_version|tcp_port'
```

The reported TCP port must be `6379`.

```bash
# Check Redis logs after replacing the placeholder with the confirmed service unit
systemctl status '<confirmed-redis-service-name>'
journalctl -u '<confirmed-redis-service-name>' --no-pager -n 100
```

```bash
# Confirm the Memcached service name from the selected external implementation
systemctl list-unit-files | grep -Ei 'memcached'
systemctl list-units --type=service --all | grep -Ei 'memcached'
```

```bash
# Verify the Memcached instance after confirming its port
systemctl status '<confirmed-memcached-service-name>'
ss -ltnp | grep ':<confirmed-memcached-port>'
```

Do not substitute an assumed Memcached port, package name, or service name until the external dependency implementation or target deployment specification confirms the values.