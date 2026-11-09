---
source-path: cookbooks/nginx-multisite
---

# Migration Plan: nginx-multisite

**TLDR**: This cookbook configures Nginx for three HTTPS-enabled virtual hosts: `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`. It installs Nginx, Fail2ban, UFW, OpenSSL, and CA certificates; creates site document roots and static index files; generates self-signed TLS certificates; deploys Nginx virtual-host configurations; enables SSH, HTTP, and HTTPS through UFW; deploys security and SSH-hardening configuration; and removes the default Nginx site. The sysctl security configuration is deployed but is not automatically applied by the original recipe because its reload resource has action `nothing` and no shown notification.

## Service Type and Instances

**Service Type**: Web Server

**Configured Instances**:

- **test.cluster.local**: Nginx HTTPS virtual host
  - Location/Path: `/opt/server/test`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Key Config: `/etc/nginx/sites-available/test.cluster.local`; enabled link at `/etc/nginx/sites-enabled/test.cluster.local`; certificate `/etc/ssl/certs/test.cluster.local.crt`; private key `/etc/ssl/private/test.cluster.local.key`; index `/opt/server/test/index.html`; SSL enabled

- **ci.cluster.local**: Nginx HTTPS virtual host
  - Location/Path: `/opt/server/ci`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Key Config: `/etc/nginx/sites-available/ci.cluster.local`; enabled link at `/etc/nginx/sites-enabled/ci.cluster.local`; certificate `/etc/ssl/certs/ci.cluster.local.crt`; private key `/etc/ssl/private/ci.cluster.local.key`; index `/opt/server/ci/index.html`; SSL enabled

- **status.cluster.local**: Nginx HTTPS virtual host
  - Location/Path: `/opt/server/status`
  - Port/Socket: HTTP `80`, HTTPS `443`
  - Key Config: `/etc/nginx/sites-available/status.cluster.local`; enabled link at `/etc/nginx/sites-enabled/status.cluster.local`; certificate `/etc/ssl/certs/status.cluster.local.crt`; private key `/etc/ssl/private/status.cluster.local.key`; index `/opt/server/status/index.html`; SSL enabled

The security collection contains Fail2ban, UFW, and SSH-hardening operations implemented by the explicit resources in `cookbooks/nginx-multisite/recipes/security.rb`.

## File Structure

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

No provider files are used by the execution tree.

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

No static files under `files/default/` or `files/` were listed. The recipes use `cookbook_file` resources for site index files, but those source files were not included in the supplied directory listing.

## Module Explanation

The cookbook entry recipe includes the operational recipes in this order:

1. `cookbooks/nginx-multisite/recipes/security.rb`
2. `cookbooks/nginx-multisite/recipes/nginx.rb`
3. `cookbooks/nginx-multisite/recipes/ssl.rb`
4. `cookbooks/nginx-multisite/recipes/sites.rb`

1. **Default entry recipe** (`cookbooks/nginx-multisite/recipes/default.rb`):
   - Includes `nginx-multisite::security`.
   - Includes `nginx-multisite::nginx`.
   - Includes `nginx-multisite::ssl`.
   - Includes `nginx-multisite::sites`.
   - Declares four `include_recipe` resources and no direct package, service, template, or file resources.

2. **Security configuration** (`cookbooks/nginx-multisite/recipes/security.rb`):
   - Installs the `fail2ban` and `ufw` packages.
   - Enables and starts the `fail2ban` service.
   - Renders `cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb` to `/etc/fail2ban/jail.local` with mode `0644`.
   - Sets the UFW default incoming policy to deny.
   - Allows SSH, HTTP, and HTTPS through UFW.
   - Enables UFW.
   - Renders `cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb` to `/etc/sysctl.d/99-security.conf` with mode `0644`.
   - Declares `execute[reload_sysctl]` with action `nothing`; no notification or subscription invoking it is shown, so the original recipe does not automatically apply the sysctl settings.
   - Conditionally declares SSH commands based on `node['security']['ssh']['disable_root']` and `node['security']['ssh']['password_auth'] == false`. The supplied analysis does not confirm the default values, so execution of these commands must be verified against `cookbooks/nginx-multisite/attributes/default.rb`.
   - The root SSH command replaces `PermitRootLogin` with `PermitRootLogin no` when enabled.
   - The password-authentication command replaces `PasswordAuthentication` with `PasswordAuthentication no` when enabled.
   - Declares the `ssh` service with action `nothing`; no restart or reload notification is shown.

3. **Nginx installation and document roots** (`cookbooks/nginx-multisite/recipes/nginx.rb`):
   - Installs the `nginx` package without a specified version.
   - Renders `cookbooks/nginx-multisite/templates/default/nginx.conf.erb` to `/etc/nginx/nginx.conf` with mode `0644`.
   - Renders `cookbooks/nginx-multisite/templates/default/security.conf.erb` to `/etc/nginx/conf.d/security.conf` with mode `0644`.
   - Enables and starts the `nginx` service.
   - Iterates over the three configured sites and creates each document root with mode `0755`.
   - Deploys one static index file per site with `cookbook_file` and mode `0644`.

   **Iterations: `test.cluster.local`**
   - Creates `/opt/server/test`.
   - Deploys `/opt/server/test/index.html`.

   **Iterations: `ci.cluster.local`**
   - Creates `/opt/server/ci`.
   - Deploys `/opt/server/ci/index.html`.

   **Iterations: `status.cluster.local`**
   - Creates `/opt/server/status`.
   - Deploys `/opt/server/status/index.html`.

   - Resource totals: one package, two templates, one service, three directories, and three cookbook files.

4. **SSL packages, directories, and certificates** (`cookbooks/nginx-multisite/recipes/ssl.rb`):
   - Installs `openssl` and `ca-certificates`.
   - Creates the `ssl-cert` group.
   - Creates `/etc/ssl/certs` with mode `0755`.
   - Creates `/etc/ssl/private` with mode `0710`.
   - Iterates over all three sites; each has `ssl_enabled: true`, so certificate generation runs for every site.
   - Generates self-signed RSA 2048-bit certificates valid for 365 days.
   - Uses the subject format `/C=US/ST=Example/L=Example/O=Example Org/OU=IT/CN=<site_name>/emailAddress=admin@example.com`.
   - Sets each private key to mode `640` and ownership `root:ssl-cert`.

   **Iterations: `test.cluster.local`**
   - Generates `/etc/ssl/certs/test.cluster.local.crt`.
   - Generates `/etc/ssl/private/test.cluster.local.key`.
   - Uses CN `test.cluster.local`.

   **Iterations: `ci.cluster.local`**
   - Generates `/etc/ssl/certs/ci.cluster.local.crt`.
   - Generates `/etc/ssl/private/ci.cluster.local.key`.
   - Uses CN `ci.cluster.local`.

   **Iterations: `status.cluster.local`**
   - Generates `/etc/ssl/certs/status.cluster.local.crt`.
   - Generates `/etc/ssl/private/status.cluster.local.key`.
   - Uses CN `status.cluster.local`.

5. **Nginx virtual-host configuration** (`cookbooks/nginx-multisite/recipes/sites.rb`):
   - Iterates over the three configured sites.
   - Renders `cookbooks/nginx-multisite/templates/default/site.conf.erb` to each site’s `sites-available` path with mode `0644`.
   - Creates a symbolic link for each site in `sites-enabled`.
   - Deletes `/etc/nginx/sites-enabled/default`.
   - No explicit Nginx reload or restart notification is shown. The migration should validate the configuration and reload Nginx when configuration changes are deployed.

   **Iterations: `test.cluster.local`**
   - Renders `/etc/nginx/sites-available/test.cluster.local`.
   - Uses `server_name: test.cluster.local`, `document_root: /opt/server/test`, `ssl_enabled: true`, certificate `/etc/ssl/certs/test.cluster.local.crt`, and key `/etc/ssl/private/test.cluster.local.key`.
   - Links `/etc/nginx/sites-enabled/test.cluster.local` to `/etc/nginx/sites-available/test.cluster.local`.

   **Iterations: `ci.cluster.local`**
   - Renders `/etc/nginx/sites-available/ci.cluster.local`.
   - Uses `server_name: ci.cluster.local`, `document_root: /opt/server/ci`, `ssl_enabled: true`, certificate `/etc/ssl/certs/ci.cluster.local.crt`, and key `/etc/ssl/private/ci.cluster.local.key`.
   - Links `/etc/nginx/sites-enabled/ci.cluster.local` to `/etc/nginx/sites-available/ci.cluster.local`.

   **Iterations: `status.cluster.local`**
   - Renders `/etc/nginx/sites-available/status.cluster.local`.
   - Uses `server_name: status.cluster.local`, `document_root: /opt/server/status`, `ssl_enabled: true`, certificate `/etc/ssl/certs/status.cluster.local.crt`, and key `/etc/ssl/private/status.cluster.local.key`.
   - Links `/etc/nginx/sites-enabled/status.cluster.local` to `/etc/nginx/sites-available/status.cluster.local`.

## Dependencies

**External cookbook dependencies**: None provided in the supplied metadata or execution tree.

**System package dependencies**:
- `fail2ban`
- `ufw`
- `nginx`
- `openssl`
- `ca-certificates`

**Service dependencies**:
- `fail2ban`: enabled and started
- `nginx`: enabled and started
- `ssh`: declared with action `nothing`; no explicit restart or reload is shown

**Firewall allowances**:
- SSH, TCP port `22`
- HTTP, TCP port `80`
- HTTPS, TCP port `443`

## Credentials

**Detection Summary**: No passwords, API tokens, data bags, encrypted data bags, Chef Vault items, CyberArk references, Conjur variables, or database credentials were detected. Three generated TLS private keys are sensitive local artifacts but are not retrieved from a credential store.

**Source**:
  - **Provider**: Generated locally by OpenSSL
  - **URL**: None
  - **Path**:
    - `/etc/ssl/private/test.cluster.local.key`
    - `/etc/ssl/private/ci.cluster.local.key`
    - `/etc/ssl/private/status.cluster.local.key`

### Generated TLS private keys

- **Variable(s)**: `key_file`, derived from `site_name`
- **Source file(s)**:
  - `cookbooks/nginx-multisite/recipes/ssl.rb`
  - `cookbooks/nginx-multisite/recipes/sites.rb`
- **Current storage**: Generated locally by the `openssl req` command; not stored in a data bag or vault
- **Usage context**: HTTPS private keys for `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`
- **Permissions and ownership**: Mode `640`, owner/group `root:ssl-cert`

### Certificate subject email

- **Variable(s)**: Hardcoded `emailAddress=admin@example.com`
- **Source file(s)**: `cookbooks/nginx-multisite/recipes/ssl.rb`
- **Current storage**: Hardcoded in the certificate-generation command
- **Usage context**: Subject metadata in the self-signed certificates
- **Secret status**: Not a credential; certificate identity metadata

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
- `/etc/nginx/sites-enabled/default` is absent
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

**Service endpoints to check**:
- `test.cluster.local`: HTTP `80`, HTTPS `443`
- `ci.cluster.local`: HTTP `80`, HTTPS `443`
- `status.cluster.local`: HTTP `80`, HTTPS `443`
- SSH: TCP `22`

**Templates rendered**:
- `fail2ban.jail.local.erb` → `/etc/fail2ban/jail.local`: 1 render
- `nginx.conf.erb` → `/etc/nginx/nginx.conf`: 1 render
- `security.conf.erb` → `/etc/nginx/conf.d/security.conf`: 1 render
- `sysctl-security.conf.erb` → `/etc/sysctl.d/99-security.conf`: 1 render
- `site.conf.erb` → `/etc/nginx/sites-available/test.cluster.local`: 1 render
- `site.conf.erb` → `/etc/nginx/sites-available/ci.cluster.local`: 1 render
- `site.conf.erb` → `/etc/nginx/sites-available/status.cluster.local`: 1 render

**Total template render operations**: 7

## Pre-flight checks

Run these checks after applying the Ansible role.

### Service status

```bash
systemctl status nginx
systemctl status fail2ban
systemctl is-enabled nginx
systemctl is-enabled fail2ban
nginx -t
```

Expected result: Nginx and Fail2ban are active and enabled, and `nginx -t` succeeds.

### Nginx configuration and virtual hosts

```bash
nginx -t
nginx -T | grep -E 'test\.cluster\.local|ci\.cluster\.local|status\.cluster\.local'
test ! -e /etc/nginx/sites-enabled/default
```

### `test.cluster.local`

```bash
test -d /opt/server/test
test -f /opt/server/test/index.html
test -f /etc/nginx/sites-available/test.cluster.local
test -L /etc/nginx/sites-enabled/test.cluster.local
readlink -f /etc/nginx/sites-enabled/test.cluster.local
test -f /etc/ssl/certs/test.cluster.local.crt
test -f /etc/ssl/private/test.cluster.local.key
openssl x509 -in /etc/ssl/certs/test.cluster.local.crt -noout -subject -issuer -dates
stat -c '%a %U:%G' /etc/ssl/private/test.cluster.local.key
curl -I -H 'Host: test.cluster.local' http://127.0.0.1/
curl -k -I --resolve test.cluster.local:443:127.0.0.1 https://test.cluster.local/
```

Expected result: The document root, index, virtual-host file, link, certificate, and key exist; the link resolves to `/etc/nginx/sites-available/test.cluster.local`; the certificate contains `CN=test.cluster.local`; the key is `640 root:ssl-cert`; and HTTP and HTTPS return valid responses.

### `ci.cluster.local`

```bash
test -d /opt/server/ci
test -f /opt/server/ci/index.html
test -f /etc/nginx/sites-available/ci.cluster.local
test -L /etc/nginx/sites-enabled/ci.cluster.local
readlink -f /etc/nginx/sites-enabled/ci.cluster.local
test -f /etc/ssl/certs/ci.cluster.local.crt
test -f /etc/ssl/private/ci.cluster.local.key
openssl x509 -in /etc/ssl/certs/ci.cluster.local.crt -noout -subject -issuer -dates
stat -c '%a %U:%G' /etc/ssl/private/ci.cluster.local.key
curl -I -H 'Host: ci.cluster.local' http://127.0.0.1/
curl -k -I --resolve ci.cluster.local:443:127.0.0.1 https://ci.cluster.local/
```

Expected result: The document root, index, virtual-host file, link, certificate, and key exist; the link resolves to `/etc/nginx/sites-available/ci.cluster.local`; the certificate contains `CN=ci.cluster.local`; the key is `640 root:ssl-cert`; and HTTP and HTTPS return valid responses.

### `status.cluster.local`

```bash
test -d /opt/server/status
test -f /opt/server/status/index.html
test -f /etc/nginx/sites-available/status.cluster.local
test -L /etc/nginx/sites-enabled/status.cluster.local
readlink -f /etc/nginx/sites-enabled/status.cluster.local
test -f /etc/ssl/certs/status.cluster.local.crt
test -f /etc/ssl/private/status.cluster.local.key
openssl x509 -in /etc/ssl/certs/status.cluster.local.crt -noout -subject -issuer -dates
stat -c '%a %U:%G' /etc/ssl/private/status.cluster.local.key
curl -I -H 'Host: status.cluster.local' http://127.0.0.1/
curl -k -I --resolve status.cluster.local:443:127.0.0.1 https://status.cluster.local/
```

Expected result: The document root, index, virtual-host file, link, certificate, and key exist; the link resolves to `/etc/nginx/sites-available/status.cluster.local`; the certificate contains `CN=status.cluster.local`; the key is `640 root:ssl-cert`; and HTTP and HTTPS return valid responses.

### Listening ports

```bash
ss -tlnp | grep -E ':(22|80|443)\b'
```

Expected result: SSH listens on `22`, and Nginx listens on `80` and `443`.

### UFW verification

```bash
ufw status verbose
ufw status numbered
```

Expected result: UFW is active, the default incoming policy is deny, and SSH, HTTP, and HTTPS are allowed.

### Fail2ban verification

```bash
fail2ban-client status
systemctl status fail2ban
test -f /etc/fail2ban/jail.local
```

### SSH hardening verification

```bash
sshd -t
sshd -T | grep -E 'permitrootlogin|passwordauthentication'
grep -E '^[# ]*PermitRootLogin|^[# ]*PasswordAuthentication' /etc/ssh/sshd_config
```

Expected result, if the corresponding security attributes enable these controls:

```text
permitrootlogin no
passwordauthentication no
```

The attribute values must be confirmed from `cookbooks/nginx-multisite/attributes/default.rb` before treating either command as expected to execute.

### Sysctl verification

```bash
test -f /etc/sysctl.d/99-security.conf
sysctl -p /etc/sysctl.d/99-security.conf
```

Expected result: The security sysctl file exists and applies without syntax errors. The original cookbook deploys the file but does not automatically run this command.