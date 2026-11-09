---
source-path: cookbooks/nginx-multisite
---

# Migration Plan: nginx-multisite

**TLDR**: This cookbook configures nginx with three SSL-enabled virtual hosts—`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`. It installs nginx, Fail2ban, UFW, OpenSSL, and CA certificates; configures nginx security settings, firewall rules, SSH hardening, kernel networking protections, and Fail2ban; creates self-signed certificates; deploys static site content; and redirects HTTP traffic to HTTPS. The Ansible migration must preserve the recipe execution order, all site-specific paths and settings, and the documented service notification behavior where confirmed from the source recipes.

## Service Type and Instances

**Service Type**: Web Server

**Configured Instances**:

- **test.cluster.local**: Static nginx virtual host
  - Location/Path: `/opt/server/test`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Certificate: `/etc/ssl/certs/test.cluster.local.crt`
  - Private key: `/etc/ssl/private/test.cluster.local.key`
  - Key settings: SSL enabled, HTTP-to-HTTPS redirect, HTTP/2, HSTS, security headers, gzip, and static-file serving

- **ci.cluster.local**: Static nginx virtual host
  - Location/Path: `/opt/server/ci`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Certificate: `/etc/ssl/certs/ci.cluster.local.crt`
  - Private key: `/etc/ssl/private/ci.cluster.local.key`
  - Key settings: SSL enabled, HTTP-to-HTTPS redirect, HTTP/2, HSTS, security headers, gzip, and static-file serving

- **status.cluster.local**: Static nginx virtual host
  - Location/Path: `/opt/server/status`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Certificate: `/etc/ssl/certs/status.cluster.local.crt`
  - Private key: `/etc/ssl/private/status.cluster.local.key`
  - Key settings: SSL enabled, HTTP-to-HTTPS redirect, HTTP/2, HSTS, security headers, gzip, and static-file serving

- **fail2ban**: Intrusion-prevention service
  - Location/Path: `/etc/fail2ban/jail.local`
  - Port/Socket: SSH, HTTP, and HTTPS jails
  - Key settings: Enabled, started, SSH and nginx jails configured

- **ufw**: Host firewall
  - Location/Path: UFW system configuration
  - Port/Socket: Allows SSH `22/tcp`, HTTP `80/tcp`, and HTTPS `443/tcp`
  - Key settings: Default incoming policy deny; firewall enabled

- **ssh**: SSH daemon hardening
  - Location/Path: `/etc/ssh/sshd_config`
  - Port/Socket: SSH `22`
  - Key settings: Root login disabled and password authentication disabled

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

No provider files or custom resources are used by the execution tree.

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
```text
```

The recipes deploy `cookbook_file` resources for the site index pages, but no static files are included in the supplied directory listing. The Ansible migration must provide source content for:

```text
test/index.html
ci/index.html
status/index.html
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/nginx-multisite/recipes/default.rb`):
   - Includes `security`, `nginx`, `ssl`, and `sites` in this exact order.
   - The Ansible role should include equivalent task files in the same sequence.
   - Resources used: four `include_recipe` resources.

2. **security** (`cookbooks/nginx-multisite/recipes/security.rb`):
   - Installs the `fail2ban` and `ufw` packages without pinned versions.
   - Enables and starts the `fail2ban` service.
   - Renders `cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb` to `/etc/fail2ban/jail.local` with mode `0644`.
   - Configures Fail2ban global values of bantime `3600`, findtime `600`, maxretry `3`, and backend `auto`.
   - Configures the following enabled jails:
     - SSH jail on `ssh`, using `/var/log/auth.log`, bantime `3600`, and maxretry `3`.
     - `nginx-http-auth` on `http,https`, using `/var/log/nginx/*error.log`, bantime `3600`, and maxretry `3`.
     - `nginx-limit-req` on `http,https`, using `/var/log/nginx/*error.log`, bantime `3600`, and maxretry `10`.
     - `nginx-botsearch` on `http,https`, using `/var/log/nginx/*access.log`, bantime `3600`, and maxretry `2`.
   - Sets the UFW default incoming policy to deny.
   - Allows SSH on `22/tcp`, HTTP on `80/tcp`, and HTTPS on `443/tcp`.
   - Enables UFW.
   - Renders `cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb` to `/etc/sysctl.d/99-security.conf` with mode `0644`.
   - Configures reverse-path filtering, redirect and source-routing protections, martian packet logging, ICMP protections, IPv6 disabling, and TCP SYN flood protections.
   - Defines the `reload_sysctl` command:
     ```bash
     sysctl -p /etc/sysctl.d/99-security.conf
     ```
   - Disables SSH root login by enforcing `PermitRootLogin no` in `/etc/ssh/sshd_config`.
   - Disables SSH password authentication by enforcing `PasswordAuthentication no` in `/etc/ssh/sshd_config`.
   - Declares the SSH service for restart when SSH configuration changes.
   - Notification timing for Fail2ban restart, SSH restart, and sysctl reload must be verified against the source recipe before being implemented as delayed handlers.

3. **nginx** (`cookbooks/nginx-multisite/recipes/nginx.rb`):
   - Installs the `nginx` package without a pinned version.
   - Renders `cookbooks/nginx-multisite/templates/default/nginx.conf.erb` to `/etc/nginx/nginx.conf` with mode `0644`.
   - Configures worker user `www-data`, automatic worker processes, PID file `/run/nginx.pid`, `768` worker connections, standard access and error logs, gzip, module includes, `/etc/nginx/conf.d/*.conf`, and `/etc/nginx/sites-enabled/*`.
   - Renders `cookbooks/nginx-multisite/templates/default/security.conf.erb` to `/etc/nginx/conf.d/security.conf` with mode `0644`.
   - Configures nginx version hiding, `login` and `api` rate-limit zones, restrictive client buffer and timeout values, SSL session settings, TLS `1.2` and `1.3`, the configured cipher list, and preferred server ciphers.
   - Enables and starts the nginx service.
   - Creates the following document roots, owned by `www-data:www-data` with mode `0755`:
     - `/opt/server/test`
     - `/opt/server/ci`
     - `/opt/server/status`
   - Deploys the following static files, owned by `www-data:www-data` with mode `0644`:
     - `test/index.html` to `/opt/server/test/index.html`
     - `ci/index.html` to `/opt/server/ci/index.html`
     - `status/index.html` to `/opt/server/status/index.html`
   - Iterations: the document-root and static-file resources execute for `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`.

4. **ssl** (`cookbooks/nginx-multisite/recipes/ssl.rb`):
   - Installs `openssl` and `ca-certificates` without pinned versions.
   - Creates the `ssl-cert` group.
   - Creates `/etc/ssl/certs` owned by `root:root` with mode `0755`.
   - Creates `/etc/ssl/private` owned by `root:ssl-cert` with mode `0710`.
   - Generates a 2048-bit RSA, unencrypted, 365-day self-signed certificate for each named site:
     - `test.cluster.local`:
       - Certificate: `/etc/ssl/certs/test.cluster.local.crt`
       - Private key: `/etc/ssl/private/test.cluster.local.key`
       - Common Name: `test.cluster.local`
     - `ci.cluster.local`:
       - Certificate: `/etc/ssl/certs/ci.cluster.local.crt`
       - Private key: `/etc/ssl/private/ci.cluster.local.key`
       - Common Name: `ci.cluster.local`
     - `status.cluster.local`:
       - Certificate: `/etc/ssl/certs/status.cluster.local.crt`
       - Private key: `/etc/ssl/private/status.cluster.local.key`
       - Common Name: `status.cluster.local`
   - Sets each private key to owner `root`, group `ssl-cert`, and mode `0640`.
   - Uses the documented certificate subject values: organization `Example Org`, organizational unit `IT`, email `admin@example.com`, country `US`, state `Example`, and locality `Example`.
   - Iterations: certificate-generation resources execute for `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`.
   - Certificate and key existence guards must be verified against the source recipe. The Ansible implementation must use `creates` or equivalent checks only if the source recipe confirms that both existing files prevent regeneration.
   - Notification behavior for nginx reloads must be verified against the source recipe before implementing it.

5. **sites** (`cookbooks/nginx-multisite/recipes/sites.rb`):
   - Renders `cookbooks/nginx-multisite/templates/default/site.conf.erb` with mode `0644`.
   - Creates and enables the following virtual hosts:
     - `test.cluster.local`:
       - Configuration: `/etc/nginx/sites-available/test.cluster.local`
       - Symlink: `/etc/nginx/sites-enabled/test.cluster.local`
       - Document root: `/opt/server/test`
       - Certificate: `/etc/ssl/certs/test.cluster.local.crt`
       - Private key: `/etc/ssl/private/test.cluster.local.key`
     - `ci.cluster.local`:
       - Configuration: `/etc/nginx/sites-available/ci.cluster.local`
       - Symlink: `/etc/nginx/sites-enabled/ci.cluster.local`
       - Document root: `/opt/server/ci`
       - Certificate: `/etc/ssl/certs/ci.cluster.local.crt`
       - Private key: `/etc/ssl/private/ci.cluster.local.key`
     - `status.cluster.local`:
       - Configuration: `/etc/nginx/sites-available/status.cluster.local`
       - Symlink: `/etc/nginx/sites-enabled/status.cluster.local`
       - Document root: `/opt/server/status`
       - Certificate: `/etc/ssl/certs/status.cluster.local.crt`
       - Private key: `/etc/ssl/private/status.cluster.local.key`
   - Each site configuration contains an HTTP listener on port `80`, an HTTP-to-HTTPS redirect, an HTTPS listener on port `443`, HTTP/2, the site-specific certificate and key, HSTS, security headers, gzip, and `try_files` static-file lookup.
   - Uses the following access and error log paths:
     - `test.cluster.local`: `/var/log/nginx/test.cluster.local_access.log` and `/var/log/nginx/test.cluster.local_error.log`
     - `ci.cluster.local`: `/var/log/nginx/ci.cluster.local_access.log` and `/var/log/nginx/ci.cluster.local_error.log`
     - `status.cluster.local`: `/var/log/nginx/status.cluster.local_access.log` and `/var/log/nginx/status.cluster.local_error.log`
   - Removes `/etc/nginx/sites-enabled/default`.
   - Iterations: virtual-host templates and links execute for `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`.
   - Notification behavior for site templates, links, and default-site removal must be verified against the source recipe before implementing delayed nginx reload handlers.

## Dependencies

**External cookbook dependencies**: None declared in `metadata.rb`.

**Supported platforms**:
- Ubuntu `>= 18.04`
- CentOS `>= 7.0`

**System package dependencies**:
- `fail2ban`
- `ufw`
- `nginx`
- `openssl`
- `ca-certificates`

**Service dependencies**:
- `fail2ban`: enabled and started
- `nginx`: enabled and started
- `ssh`: restarted when SSH hardening changes
- UFW: enabled through the firewall configuration

No package versions are specified.

## Credentials

**Detection Summary**: No passwords, API tokens, data bags, encrypted data bags, Chef Vault items, CyberArk references, Conjur variables, database credentials, or environment-based secrets were detected across the cookbook files.

**Source**:
  - **Provider**: None detected
  - **URL**: None
  - **Path**: None

### Self-Signed Certificate Subject Identity

- **Variable(s)**: No Chef attribute or secret variable; subject values are embedded in the certificate-generation command.
- **Source file(s)**: `cookbooks/nginx-multisite/recipes/ssl.rb`
- **Current storage**: Hardcoded non-secret certificate subject values
- **Usage context**: Development self-signed certificate generation
- **Values**:
  - Organization: `Example Org`
  - Organizational unit: `IT`
  - Email: `admin@example.com`
  - Country: `US`
  - State: `Example`
  - Locality: `Example`

Generated private keys are sensitive files even though they are not sourced from a credential store. Preserve:

- Private-key directory owner: `root`
- Private-key directory group: `ssl-cert`
- Private-key directory mode: `0710`
- Private-key owner: `root`
- Private-key group: `ssl-cert`
- Private-key mode: `0640`

## Checks for the Migration

**Files to verify**:

```text
/etc/fail2ban/jail.local
/etc/sysctl.d/99-security.conf
/etc/ssh/sshd_config
/etc/nginx/nginx.conf
/etc/nginx/conf.d/security.conf
/etc/nginx/sites-available/test.cluster.local
/etc/nginx/sites-available/ci.cluster.local
/etc/nginx/sites-available/status.cluster.local
/etc/nginx/sites-enabled/test.cluster.local
/etc/nginx/sites-enabled/ci.cluster.local
/etc/nginx/sites-enabled/status.cluster.local
/etc/nginx/sites-enabled/default
/opt/server/test
/opt/server/test/index.html
/opt/server/ci
/opt/server/ci/index.html
/opt/server/status
/opt/server/status/index.html
/etc/ssl/certs
/etc/ssl/private
/etc/ssl/certs/test.cluster.local.crt
/etc/ssl/private/test.cluster.local.key
/etc/ssl/certs/ci.cluster.local.crt
/etc/ssl/private/ci.cluster.local.key
/etc/ssl/certs/status.cluster.local.crt
/etc/ssl/private/status.cluster.local.key
/var/log/nginx/access.log
/var/log/nginx/error.log
/var/log/nginx/test.cluster.local_access.log
/var/log/nginx/test.cluster.local_error.log
/var/log/nginx/ci.cluster.local_access.log
/var/log/nginx/ci.cluster.local_error.log
/var/log/nginx/status.cluster.local_access.log
/var/log/nginx/status.cluster.local_error.log
/var/log/auth.log
```

`/etc/nginx/sites-enabled/default` must not exist after a successful migration.

**Service endpoints to check**:
- `test.cluster.local`: HTTP `80`, HTTPS `443`
- `ci.cluster.local`: HTTP `80`, HTTPS `443`
- `status.cluster.local`: HTTP `80`, HTTPS `443`
- SSH: `22`
- Nginx PID file: `/run/nginx.pid`
- UFW: active with default incoming policy deny
- Unix socket: none configured

**Templates rendered**:
- `cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb` → `/etc/fail2ban/jail.local`: 1 render
- `cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb` → `/etc/sysctl.d/99-security.conf`: 1 render
- `cookbooks/nginx-multisite/templates/default/nginx.conf.erb` → `/etc/nginx/nginx.conf`: 1 render
- `cookbooks/nginx-multisite/templates/default/security.conf.erb` → `/etc/nginx/conf.d/security.conf`: 1 render
- `cookbooks/nginx-multisite/templates/default/site.conf.erb` → `/etc/nginx/sites-available/test.cluster.local`: 1 render
- `cookbooks/nginx-multisite/templates/default/site.conf.erb` → `/etc/nginx/sites-available/ci.cluster.local`: 1 render
- `cookbooks/nginx-multisite/templates/default/site.conf.erb` → `/etc/nginx/sites-available/status.cluster.local`: 1 render

**Total template render operations**: `7`

## Pre-flight checks:

```bash
# Package availability
dpkg -l nginx fail2ban ufw openssl ca-certificates 2>/dev/null || \
rpm -q nginx fail2ban ufw openssl ca-certificates

# Service status
systemctl is-enabled nginx
systemctl is-active nginx
systemctl is-enabled fail2ban
systemctl is-active fail2ban
systemctl is-active ssh

# Nginx configuration validation
nginx -t
nginx -T | grep -E 'test.cluster.local|ci.cluster.local|status.cluster.local'

# Main and security configuration checks
grep -E 'user www-data|worker_processes auto|worker_connections 768|access_log|error_log|gzip on' \
  /etc/nginx/nginx.conf
grep -E 'server_tokens off|limit_req_zone|client_max_body_size|ssl_protocols|ssl_session_cache' \
  /etc/nginx/conf.d/security.conf

# test.cluster.local
test -d /opt/server/test
test -f /opt/server/test/index.html
test -f /etc/nginx/sites-available/test.cluster.local
test -L /etc/nginx/sites-enabled/test.cluster.local
test "$(readlink -f /etc/nginx/sites-enabled/test.cluster.local)" = \
  "/etc/nginx/sites-available/test.cluster.local"
test -f /etc/ssl/certs/test.cluster.local.crt
test -f /etc/ssl/private/test.cluster.local.key
openssl x509 -in /etc/ssl/certs/test.cluster.local.crt -noout -subject -dates
stat -c '%U:%G %a %n' /etc/ssl/private/test.cluster.local.key
curl -I -H 'Host: test.cluster.local' http://127.0.0.1/
curl -k -I --resolve test.cluster.local:443:127.0.0.1 \
  https://test.cluster.local/

# ci.cluster.local
test -d /opt/server/ci
test -f /opt/server/ci/index.html
test -f /etc/nginx/sites-available/ci.cluster.local
test -L /etc/nginx/sites-enabled/ci.cluster.local
test "$(readlink -f /etc/nginx/sites-enabled/ci.cluster.local)" = \
  "/etc/nginx/sites-available/ci.cluster.local"
test -f /etc/ssl/certs/ci.cluster.local.crt
test -f /etc/ssl/private/ci.cluster.local.key
openssl x509 -in /etc/ssl/certs/ci.cluster.local.crt -noout -subject -dates
stat -c '%U:%G %a %n' /etc/ssl/private/ci.cluster.local.key
curl -I -H 'Host: ci.cluster.local' http://127.0.0.1/
curl -k -I --resolve ci.cluster.local:443:127.0.0.1 \
  https://ci.cluster.local/

# status.cluster.local
test -d /opt/server/status
test -f /opt/server/status/index.html
test -f /etc/nginx/sites-available/status.cluster.local
test -L /etc/nginx/sites-enabled/status.cluster.local
test "$(readlink -f /etc/nginx/sites-enabled/status.cluster.local)" = \
  "/etc/nginx/sites-available/status.cluster.local"
test -f /etc/ssl/certs/status.cluster.local.crt
test -f /etc/ssl/private/status.cluster.local.key
openssl x509 -in /etc/ssl/certs/status.cluster.local.crt -noout -subject -dates
stat -c '%U:%G %a %n' /etc/ssl/private/status.cluster.local.key
curl -I -H 'Host: status.cluster.local' http://127.0.0.1/
curl -k -I --resolve status.cluster.local:443:127.0.0.1 \
  https://status.cluster.local/

# Site security headers
curl -k -I --resolve test.cluster.local:443:127.0.0.1 \
  https://test.cluster.local/ | grep -E \
  'Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options|Content-Security-Policy'
curl -k -I --resolve ci.cluster.local:443:127.0.0.1 \
  https://ci.cluster.local/ | grep -E \
  'Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options|Content-Security-Policy'
curl -k -I --resolve status.cluster.local:443:127.0.0.1 \
  https://status.cluster.local/ | grep -E \
  'Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options|Content-Security-Policy'

# Nginx listeners
ss -tlnp | grep -E ':(80|443)\b'

# Firewall
ufw status verbose
ufw status | grep -E '22/tcp|80/tcp|443/tcp'

# Fail2ban
fail2ban-client status
fail2ban-client status sshd
fail2ban-client status nginx-http-auth
fail2ban-client status nginx-limit-req
fail2ban-client status nginx-botsearch
grep -E 'bantime|findtime|maxretry|backend|enabled|logpath|filter' \
  /etc/fail2ban/jail.local

# SSH hardening
grep -E '^PermitRootLogin no$|^PasswordAuthentication no$' \
  /etc/ssh/sshd_config
sshd -t

# Sysctl settings
sysctl -p /etc/sysctl.d/99-security.conf
sysctl net.ipv4.conf.default.rp_filter
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.ipv4.tcp_synack_retries
sysctl net.ipv4.tcp_syn_retries

# Removed default site
test ! -e /etc/nginx/sites-enabled/default
```

Expected site-specific results:

- `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local` each return an HTTP `301` redirect from port `80` to an `https://` URL.
- HTTPS requests to all three names return HTTP `200` when their `index.html` files are present.
- Each certificate subject matches its exact hostname.
- Each private key reports ownership and mode as `root:ssl-cert 640`.
- Nginx listens on ports `80` and `443`.
- UFW is active with default incoming policy deny and allows ports `22/tcp`, `80/tcp`, and `443/tcp`.
- Fail2ban reports the `sshd`, `nginx-http-auth`, `nginx-limit-req`, and `nginx-botsearch` jails.
- SSH reports `PermitRootLogin no` and `PasswordAuthentication no`.
