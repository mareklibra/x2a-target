# Migration Plan: nginx-multisite

**TLDR**: This cookbook configures Nginx as a web server with multiple virtual hosts (3 sites), each with SSL enabled. It also sets up security measures including fail2ban, UFW firewall, and system hardening. The sites configured are test.cluster.local, ci.cluster.local, and status.cluster.local.

## Service Type and Instances

**Service Type**: Web Server

**Configured Instances**:

- **test.cluster.local**: Main test site
  - Location/Path: /opt/server/test
  - Port/Socket: 80 (redirect to 443), 443 (SSL)
  - Key Config: SSL enabled, document root at /opt/server/test

- **ci.cluster.local**: Continuous integration site
  - Location/Path: /opt/server/ci
  - Port/Socket: 80 (redirect to 443), 443 (SSL)
  - Key Config: SSL enabled, document root at /opt/server/ci

- **status.cluster.local**: Status monitoring site
  - Location/Path: /opt/server/status
  - Port/Socket: 80 (redirect to 443), 443 (SSL)
  - Key Config: SSL enabled, document root at /opt/server/status

## File Structure

```
cookbooks/nginx-multisite/recipes/default.rb
cookbooks/nginx-multisite/recipes/nginx.rb
cookbooks/nginx-multisite/recipes/security.rb
cookbooks/nginx-multisite/recipes/sites.rb
cookbooks/nginx-multisite/recipes/ssl.rb
cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb
cookbooks/nginx-multisite/templates/default/nginx.conf.erb
cookbooks/nginx-multisite/templates/default/security.conf.erb
cookbooks/nginx-multisite/templates/default/site.conf.erb
cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb
cookbooks/nginx-multisite/attributes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/nginx-multisite/recipes/default.rb`):
   - Includes other recipes in sequence: security, nginx, ssl, sites
   - Resources: include_recipe (4)

2. **security** (`cookbooks/nginx-multisite/recipes/security.rb`):
   - Installs security packages: fail2ban, ufw
   - Configures fail2ban with custom jail settings
   - Sets up UFW firewall with default deny policy and allows SSH, HTTP, HTTPS
   - Deploys system security settings via sysctl
   - Hardens SSH configuration by disabling root login and password authentication
   - Resources: package (1), service (1), template (1), execute (8), service (1)

3. **nginx** (`cookbooks/nginx-multisite/recipes/nginx.rb`):
   - Installs nginx package
   - Deploys main nginx.conf configuration
   - Deploys security-specific nginx configuration
   - Creates document root directories for each site
   - Deploys index.html files for each site
   - Resources: package (1), template (2), service (1), directory (3), cookbook_file (3)
   - Creates document root directory for test.cluster.local
   - Creates document root directory for ci.cluster.local
   - Creates document root directory for status.cluster.local
   - Deploys index.html file for test.cluster.local
   - Deploys index.html file for ci.cluster.local
   - Deploys index.html file for status.cluster.local

4. **ssl** (`cookbooks/nginx-multisite/recipes/ssl.rb`):
   - Installs SSL-related packages: openssl, ca-certificates
   - Creates ssl-cert group
   - Creates certificate and private key directories
   - Generates self-signed SSL certificates for each site
   - Resources: package (1), group (1), directory (2), execute (3)
   - Generates self-signed SSL certificate for test.cluster.local
   - Generates self-signed SSL certificate for ci.cluster.local
   - Generates self-signed SSL certificate for status.cluster.local

5. **sites** (`cookbooks/nginx-multisite/recipes/sites.rb`):
   - Creates Nginx virtual host configuration for each site
   - Creates symlinks from sites-available to sites-enabled
   - Removes default site configuration
   - Resources: template (3), link (3), file (1)
   - Creates site configuration in sites-available for test.cluster.local
   - Creates site configuration in sites-available for ci.cluster.local
   - Creates site configuration in sites-available for status.cluster.local
   - Creates symlink in sites-enabled for test.cluster.local
   - Creates symlink in sites-enabled for ci.cluster.local
   - Creates symlink in sites-enabled for status.cluster.local

## Dependencies

**External cookbook dependencies**: None specified in the provided information
**System package dependencies**: nginx, fail2ban, ufw, openssl, ca-certificates
**Service dependencies**: nginx, fail2ban, ssh

## Checks for the Migration

**Files to verify**:
- /etc/nginx/nginx.conf
- /etc/nginx/conf.d/security.conf
- /etc/nginx/sites-available/test.cluster.local
- /etc/nginx/sites-available/ci.cluster.local
- /etc/nginx/sites-available/status.cluster.local
- /etc/nginx/sites-enabled/test.cluster.local
- /etc/nginx/sites-enabled/ci.cluster.local
- /etc/nginx/sites-enabled/status.cluster.local
- /etc/fail2ban/jail.local
- /etc/sysctl.d/99-security.conf
- /etc/ssl/certs/test.cluster.local.crt
- /etc/ssl/certs/ci.cluster.local.crt
- /etc/ssl/certs/status.cluster.local.crt
- /etc/ssl/private/test.cluster.local.key
- /etc/ssl/private/ci.cluster.local.key
- /etc/ssl/private/status.cluster.local.key
- /opt/server/test/index.html
- /opt/server/ci/index.html
- /opt/server/status/index.html

**Service endpoints to check**:
- Ports listening: 80, 443
- Network interfaces: All interfaces (default)

**Templates rendered**:
- nginx.conf.erb → /etc/nginx/nginx.conf (1 time)
- security.conf.erb → /etc/nginx/conf.d/security.conf (1 time)
- site.conf.erb → /etc/nginx/sites-available/test.cluster.local (1 time)
- site.conf.erb → /etc/nginx/sites-available/ci.cluster.local (1 time)
- site.conf.erb → /etc/nginx/sites-available/status.cluster.local (1 time)
- fail2ban.jail.local.erb → /etc/fail2ban/jail.local (1 time)
- sysctl-security.conf.erb → /etc/sysctl.d/99-security.conf (1 time)

## Pre-flight checks:
```bash
# Service status
systemctl status nginx
systemctl status fail2ban
systemctl status ufw

# Nginx configuration validation
nginx -t

# Check Nginx processes
ps aux | grep nginx

# Check site configurations - test each site individually
# Site: test.cluster.local
curl -I http://test.cluster.local
curl -I -k https://test.cluster.local
curl -s -k https://test.cluster.local | grep -i "test"

# Site: ci.cluster.local
curl -I http://ci.cluster.local
curl -I -k https://ci.cluster.local
curl -s -k https://ci.cluster.local | grep -i "ci"

# Site: status.cluster.local
curl -I http://status.cluster.local
curl -I -k https://status.cluster.local
curl -s -k https://status.cluster.local | grep -i "status"

# SSL certificate verification
openssl x509 -in /etc/ssl/certs/test.cluster.local.crt -text -noout | grep "Subject:"
openssl x509 -in /etc/ssl/certs/ci.cluster.local.crt -text -noout | grep "Subject:"
openssl x509 -in /etc/ssl/certs/status.cluster.local.crt -text -noout | grep "Subject:"

# Check SSL configuration
nmap --script ssl-enum-ciphers -p 443 test.cluster.local
nmap --script ssl-enum-ciphers -p 443 ci.cluster.local
nmap --script ssl-enum-ciphers -p 443 status.cluster.local

# Firewall status
ufw status verbose

# Fail2ban status
fail2ban-client status
fail2ban-client status sshd
fail2ban-client status nginx-http-auth
fail2ban-client status nginx-limit-req
fail2ban-client status nginx-botsearch

# SSH hardening verification
grep "PermitRootLogin" /etc/ssh/sshd_config
grep "PasswordAuthentication" /etc/ssh/sshd_config

# Sysctl security settings
sysctl -a | grep "net.ipv4.conf.all.rp_filter"
sysctl -a | grep "net.ipv4.conf.all.accept_redirects"
sysctl -a | grep "net.ipv4.tcp_syncookies"

# Document root permissions
ls -la /opt/server/test/
ls -la /opt/server/ci/
ls -la /opt/server/status/

# Network listening
netstat -tulpn | grep nginx
ss -tlnp | grep nginx
lsof -i :80
lsof -i :443

# Log files
tail -n 50 /var/log/nginx/access.log
tail -n 50 /var/log/nginx/error.log
tail -n 50 /var/log/nginx/test.cluster.local_access.log
tail -n 50 /var/log/nginx/ci.cluster.local_access.log
tail -n 50 /var/log/nginx/status.cluster.local_access.log
```