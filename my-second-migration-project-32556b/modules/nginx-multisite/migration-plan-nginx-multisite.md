---
source-path: cookbooks/nginx-multisite
---

# Migration Plan: nginx-multisite

**TLDR**: This cookbook configures three nginx sites—`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`—with site-specific document roots, SSL settings, virtual-host templates, static index files, and self-signed certificates. It installs nginx, fail2ban, UFW, OpenSSL, and CA certificates; configures security resources; creates SSL directories and certificates; deploys site content; renders virtual-host configurations; and enables the configured services. Template contents, resource ownership and modes, platform support, dependency metadata, and notification relationships require confirmation from the cookbook source before being treated as verified migration behavior.

## Service Type and Instances

**Service Type**: Web Server

**Configured Instances**:

- **test.cluster.local**: nginx site
  - Location/Path: `/opt/server/test`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Key Config: SSL enabled; certificate `/etc/ssl/certs/test.cluster.local.crt`; private key `/etc/ssl/private/test.cluster.local.key`; source file `test/index.html`; deployed file `/opt/server/test/index.html`

- **ci.cluster.local**: nginx site
  - Location/Path: `/opt/server/ci`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Key Config: SSL enabled; certificate `/etc/ssl/certs/ci.cluster.local.crt`; private key `/etc/ssl/private/ci.cluster.local.key`; source file `ci/index.html`; deployed file `/opt/server/ci/index.html`

- **status.cluster.local**: nginx site
  - Location/Path: `/opt/server/status`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Key Config: SSL enabled; certificate `/etc/ssl/certs/status.cluster.local.crt`; private key `/etc/ssl/private/status.cluster.local.key`; source file `status/index.html`; deployed file `/opt/server/status/index.html`

**Configured security settings**:

- **fail2ban**: enabled by default
- **ufw**: enabled by default
- **ssh**:
  - `disable_root: true`
  - `password_auth: false`

## File Structure

**MANDATORY: Preserve this section from the original plan.**

**Recipes:**
```text
cookbooks/nginx-multisite/recipes/default.rb
cookbooks/nginx-multisite/recipes/security.rb
cookbooks/nginx-multisite/recipes/nginx.rb
cookbooks/nginx-multisite/recipes/ssl.rb
cookbooks/nginx-multisite/recipes/sites.rb
```

**Providers:**
```text
```

No custom-resource provider is used by the execution tree.

**Templates:**
```text
cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb
cookbooks/nginx-multisite/templates/default/nginx.conf.erb
cookbooks/nginx-multisite/templates/default/security.conf.erb
cookbooks/nginx-multisite/templates/default/site.conf.erb
cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb
```

**Attributes:**
```text
cookbooks/nginx-multisite/attributes/default.rb
```

**Files:**

No static files are included in the supplied file listing. The `nginx.rb` recipe references these `cookbook_file` sources:

```text
test/index.html
ci/index.html
status/index.html
```

These files must be supplied in the Ansible role or project if the static landing pages are required.

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/nginx-multisite/recipes/default.rb`):
   - Establishes the cookbook execution flow.
   - Executes the `security`, `nginx`, `ssl`, and `sites` recipes in that order.

2. **security** (`cookbooks/nginx-multisite/recipes/security.rb`):
   - Installs the `fail2ban` and `ufw` packages.
   - Manages the `fail2ban` service.
   - Renders `cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb` to `/etc/fail2ban/jail.local`.
   - Applies UFW commands for the default policy and SSH, HTTP, and HTTPS access.
   - Renders `cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb` to `/etc/sysctl.d/99-security.conf`.
   - Declares the `reload_sysctl` execute resource with `action: nothing`.
   - Applies conditional SSH hardening for:
     - `PermitRootLogin no`
     - `PasswordAuthentication no`
   - Declares the `ssh` service for conditional restart behavior.
   - Resource counts: 1 package, 2 services, 2 templates, 8 execute resources, and 2 conditional execute resources.
   - Confirmed template contents, command guards, ownership, modes, and notification relationships require source verification before migration.

3. **nginx** (`cookbooks/nginx-multisite/recipes/nginx.rb`):
   - Installs the `nginx` package.
   - Renders `cookbooks/nginx-multisite/templates/default/nginx.conf.erb` to `/etc/nginx/nginx.conf`.
   - Renders `cookbooks/nginx-multisite/templates/default/security.conf.erb` to `/etc/nginx/conf.d/security.conf`.
   - Manages the nginx service.
   - Creates the document root for **test.cluster.local** at `/opt/server/test`.
   - Creates the document root for **ci.cluster.local** at `/opt/server/ci`.
   - Creates the document root for **status.cluster.local** at `/opt/server/status`.
   - Deploys `test/index.html` to `/opt/server/test/index.html`.
   - Deploys `ci/index.html` to `/opt/server/ci/index.html`.
   - Deploys `status/index.html` to `/opt/server/status/index.html`.
   - Resource counts: 1 package, 2 templates, 1 service, 3 directories, and 3 `cookbook_file` resources.
   - Confirmed template contents, file ownership, modes, and notification relationships require source verification before migration.

4. **ssl** (`cookbooks/nginx-multisite/recipes/ssl.rb`):
   - Installs the `openssl` and `ca-certificates` packages.
   - Creates the `ssl-cert` group.
   - Creates `/etc/ssl/certs`.
   - Creates `/etc/ssl/private`.
   - Generates a self-signed certificate and private key for **test.cluster.local**:
     - Certificate: `/etc/ssl/certs/test.cluster.local.crt`
     - Key: `/etc/ssl/private/test.cluster.local.key`
     - Common Name: `test.cluster.local`
   - Generates a self-signed certificate and private key for **ci.cluster.local**:
     - Certificate: `/etc/ssl/certs/ci.cluster.local.crt`
     - Key: `/etc/ssl/private/ci.cluster.local.key`
     - Common Name: `ci.cluster.local`
   - Generates a self-signed certificate and private key for **status.cluster.local**:
     - Certificate: `/etc/ssl/certs/status.cluster.local.crt`
     - Key: `/etc/ssl/private/status.cluster.local.key`
     - Common Name: `status.cluster.local`
   - Resource counts: 1 package, 1 group, 2 directories, and 3 execute resources.
   - Certificate command details, file permissions, ownership, and notification relationships require source verification before migration.

5. **sites** (`cookbooks/nginx-multisite/recipes/sites.rb`):
   - Renders `cookbooks/nginx-multisite/templates/default/site.conf.erb` for **test.cluster.local** to `/etc/nginx/sites-available/test.cluster.local`.
   - Creates `/etc/nginx/sites-enabled/test.cluster.local` pointing to `/etc/nginx/sites-available/test.cluster.local`.
   - Renders `cookbooks/nginx-multisite/templates/default/site.conf.erb` for **ci.cluster.local** to `/etc/nginx/sites-available/ci.cluster.local`.
   - Creates `/etc/nginx/sites-enabled/ci.cluster.local` pointing to `/etc/nginx/sites-available/ci.cluster.local`.
   - Renders `cookbooks/nginx-multisite/templates/default/site.conf.erb` for **status.cluster.local** to `/etc/nginx/sites-available/status.cluster.local`.
   - Creates `/etc/nginx/sites-enabled/status.cluster.local` pointing to `/etc/nginx/sites-available/status.cluster.local`.
   - Removes `/etc/nginx/sites-enabled/default`.
   - Resource counts: 3 templates, 3 links, and 1 file resource.
   - Confirmed virtual-host template contents and notification relationships require source verification before migration.

## Dependencies

**External cookbook dependencies**: Not confirmed by the supplied analysis; verify cookbook metadata before migration.

**System package dependencies**:

- `nginx`
- `fail2ban`
- `ufw`
- `openssl`
- `ca-certificates`

**Service dependencies**:

- `fail2ban`
- `nginx`
- `ssh`

The supplied analysis confirms resources for these services. Enabled, started, restart, and reload behavior should be verified against the recipe source and target operating system.

## Credentials

**Detection Summary**: No credentials or secret material were detected.

**Source**:
  - **Provider**: None detected
  - **URL**: None
  - **Path**: None

No data bags, encrypted data bags, Chef Vault items, CyberArk integrations, Conjur variables, Vault attributes, API keys, database credentials, passwords, or secret environment variables were detected. The cookbook creates self-signed TLS certificates, but these are development certificates rather than stored credentials.

No credentials or secrets were detected in this cookbook. All analyzed configuration values appear to be non-sensitive.

## Checks for the Migration

**Files to verify**:

- `/etc/nginx/nginx.conf`
- `/etc/nginx/conf.d/security.conf`
- `/etc/nginx/sites-available/test.cluster.local`
- `/etc/nginx/sites-available/ci.cluster.local`
- `/etc/nginx/sites-available/status.cluster.local`
- `/etc/nginx/sites-enabled/test.cluster.local`
- `/etc/nginx/sites-enabled/ci.cluster.local`
- `/etc/nginx/sites-enabled/status.cluster.local`
- `/etc/fail2ban/jail.local`
- `/etc/sysctl.d/99-security.conf`
- `/etc/ssh/sshd_config`
- `/opt/server/test/index.html`
- `/opt/server/ci/index.html`
- `/opt/server/status/index.html`
- `/etc/ssl/certs/test.cluster.local.crt`
- `/etc/ssl/private/test.cluster.local.key`
- `/etc/ssl/certs/ci.cluster.local.crt`
- `/etc/ssl/private/ci.cluster.local.key`
- `/etc/ssl/certs/status.cluster.local.crt`
- `/etc/ssl/private/status.cluster.local.key`
- `/var/log/nginx/access.log`
- `/var/log/nginx/error.log`
- `/var/log/nginx/test.cluster.local_access.log`
- `/var/log/nginx/test.cluster.local_error.log`
- `/var/log/nginx/ci.cluster.local_access.log`
- `/var/log/nginx/ci.cluster.local_error.log`
- `/var/log/nginx/status.cluster.local_access.log`
- `/var/log/nginx/status.cluster.local_error.log`

**Service endpoints to check**:

- SSH: TCP `22`
- HTTP: TCP `80`
- HTTPS: TCP `443`

**Unix sockets and IPC**:

- No application-specific Unix socket is configured.
- nginx PID file: `/run/nginx.pid`

**Templates rendered**:

- `fail2ban.jail.local.erb` → `/etc/fail2ban/jail.local`: 1 render
- `nginx.conf.erb` → `/etc/nginx/nginx.conf`: 1 render
- `security.conf.erb` → `/etc/nginx/conf.d/security.conf`: 1 render
- `sysctl-security.conf.erb` → `/etc/sysctl.d/99-security.conf`: 1 render
- `site.conf.erb` → `/etc/nginx/sites-available/test.cluster.local`: 1 render
- `site.conf.erb` → `/etc/nginx/sites-available/ci.cluster.local`: 1 render
- `site.conf.erb` → `/etc/nginx/sites-available/status.cluster.local`: 1 render
- Total `site.conf.erb` renders: 3

## Pre-flight checks:

```bash
# Service status
systemctl status nginx
systemctl status fail2ban
systemctl status ssh

# nginx configuration and enabled sites
nginx -t
nginx -T | grep -E 'test\.cluster\.local|ci\.cluster\.local|status\.cluster\.local'
test -L /etc/nginx/sites-enabled/test.cluster.local
test -L /etc/nginx/sites-enabled/ci.cluster.local
test -L /etc/nginx/sites-enabled/status.cluster.local

# test.cluster.local
curl -I -H 'Host: test.cluster.local' http://127.0.0.1/
curl -k -I --resolve test.cluster.local:443:127.0.0.1 https://test.cluster.local/
curl -k --resolve test.cluster.local:443:127.0.0.1 https://test.cluster.local/
test -d /opt/server/test
test -f /opt/server/test/index.html
test -f /etc/ssl/certs/test.cluster.local.crt
test -f /etc/ssl/private/test.cluster.local.key
test -f /var/log/nginx/test.cluster.local_access.log
test -f /var/log/nginx/test.cluster.local_error.log

# ci.cluster.local
curl -I -H 'Host: ci.cluster.local' http://127.0.0.1/
curl -k -I --resolve ci.cluster.local:443:127.0.0.1 https://ci.cluster.local/
curl -k --resolve ci.cluster.local:443:127.0.0.1 https://ci.cluster.local/
test -d /opt/server/ci
test -f /opt/server/ci/index.html
test -f /etc/ssl/certs/ci.cluster.local.crt
test -f /etc/ssl/private/ci.cluster.local.key
test -f /var/log/nginx/ci.cluster.local_access.log
test -f /var/log/nginx/ci.cluster.local_error.log

# status.cluster.local
curl -I -H 'Host: status.cluster.local' http://127.0.0.1/
curl -k -I --resolve status.cluster.local:443:127.0.0.1 https://status.cluster.local/
curl -k --resolve status.cluster.local:443:127.0.0.1 https://status.cluster.local/
test -d /opt/server/status
test -f /opt/server/status/index.html
test -f /etc/ssl/certs/status.cluster.local.crt
test -f /etc/ssl/private/status.cluster.local.key
test -f /var/log/nginx/status.cluster.local_access.log
test -f /var/log/nginx/status.cluster.local_error.log

# TLS certificates
openssl x509 -in /etc/ssl/certs/test.cluster.local.crt -noout -subject -dates
openssl x509 -in /etc/ssl/certs/ci.cluster.local.crt -noout -subject -dates
openssl x509 -in /etc/ssl/certs/status.cluster.local.crt -noout -subject -dates

# Firewall
ufw status verbose
ufw status numbered

# SSH configuration
grep -E '^(PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config
sshd -t

# fail2ban
fail2ban-client status
fail2ban-client status sshd
fail2ban-client status nginx-http-auth
fail2ban-client status nginx-limit-req
fail2ban-client status nginx-botsearch

# Kernel settings
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.default.rp_filter
sysctl net.ipv4.conf.all.accept_redirects
sysctl net.ipv4.conf.all.send_redirects
sysctl net.ipv4.conf.all.accept_source_route
sysctl net.ipv4.icmp_echo_ignore_all
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.ipv6.conf.all.disable_ipv6

# Listening ports
ss -tlnp | grep ':22 '
ss -tlnp | grep ':80 '
ss -tlnp | grep ':443 '

# nginx logs
tail -n 50 /var/log/nginx/access.log
tail -n 50 /var/log/nginx/error.log
tail -n 50 /var/log/nginx/test.cluster.local_access.log
tail -n 50 /var/log/nginx/test.cluster.local_error.log
tail -n 50 /var/log/nginx/ci.cluster.local_access.log
tail -n 50 /var/log/nginx/ci.cluster.local_error.log
tail -n 50 /var/log/nginx/status.cluster.local_access.log
tail -n 50 /var/log/nginx/status.cluster.local_error.log
```

Expected endpoint behavior for **test.cluster.local**:

- HTTP returns `301`.
- The redirect target uses `https://test.cluster.local/`.
- HTTPS returns `200 OK`.
- The response body matches `/opt/server/test/index.html`.

Expected endpoint behavior for **ci.cluster.local**:

- HTTP returns `301`.
- The redirect target uses `https://ci.cluster.local/`.
- HTTPS returns `200 OK`.
- The response body matches `/opt/server/ci/index.html`.

Expected endpoint behavior for **status.cluster.local**:

- HTTP returns `301`.
- The redirect target uses `https://status.cluster.local/`.
- HTTPS returns `200 OK`.
- The response body matches `/opt/server/status/index.html`.

Confirm the following before accepting the migration:

- nginx, fail2ban, and SSH have the expected service state.
- nginx validates successfully.
- All three named site links and configuration files exist.
- All three document roots and static files exist.
- All three certificate files exist and identify their corresponding site names.
- UFW is active with the required SSH, HTTP, and HTTPS access.
- SSH configuration passes `sshd -t`.
- The configured fail2ban resources are available.
- The configured network ports are listening.
- Site-specific nginx logs are created and contain requests for their corresponding sites.
- Template contents, resource properties, platform support, external cookbook dependencies, and notification behavior have been verified against the cookbook source where the supplied analysis did not confirm them.