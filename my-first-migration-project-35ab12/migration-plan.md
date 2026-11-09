# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant deployment with three local cookbooks: an Nginx multi-site web tier, caching services, and a FastAPI tutorial application backed by PostgreSQL. The migration is moderate rather than large: the cookbook count is low, but the deployment combines web serving, TLS, host hardening, firewalling, Redis/Memcached, PostgreSQL, Python packaging, and systemd. A realistic initial migration is approximately 2–4 weeks for one engineer familiar with Ansible, including implementation, validation, and security review; production hardening and certificate/secrets integration may extend that schedule.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server configuration for three static virtual hosts (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), with HTTP-to-HTTPS redirects, self-signed TLS certificates, security headers, compression, per-site logging, and host hardening.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Nginx installation and service management; generated site configurations; certificate/key directories; self-signed RSA certificates; fail2ban; UFW; SSH hardening; sysctl network protections; static HTML content.

- **cache**:
    - Description: Memcached and Redis service provisioning, including Redis logging, a password-protected Redis instance on port 6379, service enablement, and a post-generation Redis configuration workaround.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: `memcached` cookbook inclusion; `redisio` cookbook inclusion; Redis authentication; `/var/log/redis`; configuration cleanup via Ruby block; Redis enablement.

- **fastapi-tutorial**:
    - Description: Deployment of the external FastAPI tutorial application with Python 3, a virtual environment, PostgreSQL, an application database/user, environment configuration, and a root-run systemd service using Uvicorn on port 8000.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Git checkout from `https://github.com/dibanez/fastapi_tutorial.git` at `main`; Python dependency installation; PostgreSQL service; `fastapi` database/user; `.env` file; systemd unit and daemon reload.

**CRITICAL PATH VERIFICATION:**
All module paths above were present in the supplied repository tree and their primary `metadata.rb` and `recipes/default.rb` files were reviewed. The `nginx-multisite` split recipes were also reviewed because the default recipe explicitly includes them.

### Infrastructure Files

- `Berksfile`: Chef Supermarket source, local cookbook declarations, and external dependencies. In Ansible, replace cookbook resolution with `collections/requirements.yml`, role dependencies, OS package repositories, and pinned automation content.
- `Vagrantfile`: Defines a `generic/fedora42` VM, libvirt provider, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, ports 80/443 forwarded to 8080/8443, and rsync of the repository to `/chef-repo`. Replace the Chef-oriented provisioner with `ansible_local` or host-side Ansible; reconcile the Fedora image with the cookbook's Debian-oriented commands.
- `solo.json`: Chef run list, site definitions, TLS paths, and security attributes. Convert to group/host variables and role defaults/vars, with secrets removed from plaintext inventory data.
- `solo.rb`: Chef Solo cache, cookbook paths, logging, and STDOUT behavior. It has no direct Ansible equivalent beyond project layout and callback/logging configuration.
- `vagrant-provision.sh`: Installs packages, Chef, Berkshelf, dependencies, and runs Chef Solo. Replace with Ansible installation/bootstrap tooling and a repeatable playbook invocation; remove the remote `curl | bash` installer pattern.
- `cookbooks/nginx-multisite/templates/`: Nginx, site, security, fail2ban, and sysctl templates. Convert ERB variables/conditionals to Jinja2 templates.
- `cookbooks/nginx-multisite/files/`: Static `index.html` files for the three sites. Move to role `files/` and deploy based on a site list.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Present in the tree but not referenced by the reviewed default recipes; assess whether it is dead code before migration.

### Target Details

- **Operating System**: The Vagrant target is explicitly Fedora 42 (`generic/fedora42`), while the Chef cookbooks declare support for Ubuntu >=18.04 and CentOS >=7 and use Debian-specific commands/packages (`apt-get`, `ufw`, `www-data`, `/var/log/auth.log`, and the `ssh` service name). The effective tested target is therefore ambiguous and requires a decision. Prefer a supported, explicitly tested OS such as Ubuntu 22.04/24.04 for behavioral parity, or create Fedora-specific Ansible variables/tasks and use `firewalld`, `dnf`, Fedora users/groups, and the correct SSH journal/log handling.
- **Virtual Machine Technology**: libvirt via Vagrant, with rsync-backed source synchronization.
- **Cloud Platform**: Not specified; no AWS, Azure, or GCP configuration was found.

## Migration Approach

Create an Ansible project with separate roles for `nginx_multisite`, `cache`, and `fastapi_tutorial`, plus a shared hardening role or carefully scoped hardening tasks. Use a site data structure equivalent to `nginx.sites`, handlers for Nginx/fail2ban/sysctl/systemd changes, `ansible.builtin.package`, `service`, `template`, `copy`, `file`, `systemd`, `git`, `pip`, `postgresql_*`, and `community.general.ufw` or an OS-native firewall module. Add Molecule or equivalent VM tests and run the playbook against the Vagrant VM in check mode before applying changes.

### Key Dependencies to Address

- **Chef >=16 / Chef Solo / Berkshelf**: Remove entirely; use `ansible-core` with a pinned version and `collections/requirements.yml`.
- **nginx cookbook `~> 12.0`**: Replace with an internal Ansible role or `nginxinc.nginx` only after confirming module behavior and supported OS versions. Preserve the reviewed custom templates rather than assuming cookbook defaults.
- **memcached cookbook `~> 6.0`**: Replace with package/service tasks and explicit configuration, or a pinned Ansible collection/role.
- **redisio cookbook `~> 7.2.4`**: Replace with a tested Redis role or native package/configuration tasks. Re-evaluate the Ruby post-processing workaround instead of porting it literally.
- **PostgreSQL packages and client tools**: Use distribution packages and `community.postgresql` modules for database/user/grants, with idempotent existence checks.
- **External Git repository and Python requirements**: Pin the repository to a commit or release rather than `main`; install dependencies in the virtual environment using a lock/constraints strategy.
- **OpenSSL, CA certificates, fail2ban, UFW, SSH, sysctl**: Replace with Ansible package, template, module, and handler implementations, accounting for OS-specific firewall and service names.

### Security Considerations

- **Plaintext Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123` in source. Move it to Ansible Vault or an external secret manager, rotate it, and ensure Redis configuration and logs do not expose it.
- **Plaintext PostgreSQL credentials**: The FastAPI recipe creates user `fastapi` with password `fastapi_password` and writes the same password to a world-readable (`0644`) `.env` file and database URL. Treat the credential as compromised, rotate it, store it in Vault, use restrictive ownership/mode (for example 0600), and avoid shell command interpolation.
- **TLS certificates and keys**: The Nginx role creates 365-day self-signed certificates under `/etc/ssl/certs` and `/etc/ssl/private`; private keys are set to group `ssl-cert` and mode 0640. This is appropriate only for development. Production should use managed certificates, explicit renewal, protected key material, and validation of certificate ownership/permissions. Do not commit generated keys.
- **SSH hardening**: Root login and password authentication are disabled through `sed` edits. Implement managed `sshd_config` drop-ins or validated line edits, preserve an out-of-band key-based access path, and use `validate: sshd -t` before restart.
- **Firewall and service exposure**: UFW defaults to deny and allows SSH/HTTP/HTTPS, but the FastAPI service listens on all interfaces at port 8000 and Redis/Memcached behavior is not explicitly firewalled. Define intended bind addresses and firewall rules; do not expose databases/caches unnecessarily.
- **Nginx security controls**: Preserve TLS 1.2/1.3, HSTS, security headers, request limits, timeouts, hidden version information, and fail2ban jails, but review the custom cipher list, CSP use of `unsafe-inline`, 1 KB request limits, and compatibility before production rollout.
- **System hardening**: The sysctl template disables IPv6 and ICMP responses and enables reverse-path filtering/SYN cookies. Validate these settings against networking, monitoring, container, and IPv6 requirements before applying globally.
- **Supply chain and installer risk**: The provisioning script downloads Chef via `curl` and installs dependencies from an unpinned external Git branch and pip requirements. Pin and verify all artifacts in the Ansible implementation.

### Technical Challenges

- **OS mismatch**: Vagrant provisions Fedora 42 but the script starts with `apt-get`, and cookbook defaults assume Ubuntu paths/users. Resolve the target OS first; maintain separate variable/task files if Fedora remains required.
- **Idempotency of database setup**: Chef uses `psql` shell commands with `|| true`, which can hide failures. Use PostgreSQL modules, explicit database/user privileges, and secure password handling.
- **TLS lifecycle**: Self-signed generation is conditional only on file existence and has no renewal or SAN strategy. Define development versus production certificate ownership and ensure certificates include appropriate SANs.
- **Nginx template correctness**: The site template generates separate HTTP/HTTPS server blocks and uses dynamic site data; test rendered configs with `nginx -t` before reload. Confirm document roots: cookbook attributes use `/opt/server/...`, while `solo.json` overrides them to `/var/www/...`.
- **Chef data precedence conversion**: `solo.json` overrides cookbook defaults and carries the actual three-site configuration. Model this explicitly in inventory/group vars and document precedence so defaults do not silently win.
- **Redis workaround behavior**: The Ruby block deletes replication-related lines from generated config. Determine why the dependency emits them and replace the workaround with supported Redis settings and a tested template.
- **Application service design**: The service runs Uvicorn as root and installs from a mutable branch. Introduce a dedicated unprivileged user, pinned application revision/dependencies, environment file permissions, health checks, and a controlled restart policy.
- **Testing and connectivity**: Validate virtual hosts, local DNS/hosts entries, forwarded ports, HTTPS warnings, application-to-PostgreSQL connectivity, and cache availability in the Vagrant/libvirt environment.

### Migration Order

1. **Foundation and target decision**: Pin Ansible/collections, settle Ubuntu versus Fedora, define inventory, Vault policy, testing conventions, and Vagrant/Ansible provisioning.
2. **cache**: Migrate Memcached and Redis package/service configuration, secret handling, bind/firewall policy, and idempotent verification. This is relatively isolated but contains credential risk.
3. **fastapi-tutorial**: Migrate OS packages, PostgreSQL database objects, application checkout/venv, secure environment file, and systemd unit. Validate application startup and database connectivity.
4. **nginx-multisite base and content**: Implement Nginx installation, static content, virtual host data model, templates, handlers, and HTTP/HTTPS behavior.
5. **nginx-multisite security controls**: Add TLS provisioning/installation, fail2ban, firewall, SSH hardening, sysctl, and security headers after access recovery and rollback procedures are tested.
6. **Integration and cutover**: Run parallel validation against Vagrant, perform security review, compare rendered configs/service states, document operations, and retire Chef/Berkshelf only after rollback criteria are met.

### Assumptions

- The supplied tree is complete and contains no hidden Chef roles, tests, encrypted data bags, environments, or production certificate files.
- `solo.json` is the intended runtime configuration and overrides the Nginx cookbook's `/opt/server/...` default roots with `/var/www/...`.
- The three static HTML files are the complete site payload for this repository; the external FastAPI repository is not being migrated as application source.
- `generic/fedora42` is the current local test image, but the use of `apt-get`, UFW, `www-data`, and Debian log paths may indicate the configuration was originally designed for Ubuntu. This must be confirmed before implementation.
- The target Ansible control node and managed-node connectivity, Python availability, privilege escalation method, and supported Ansible version are not specified.
- No cloud provider, CI/CD system, production inventory, DNS automation, certificate authority, backup policy, monitoring integration, or secret manager is present in the reviewed files.
- The external cookbook versions are declared in Berksfile, but their exact resolved versions and behavior are not captured; compatibility must be verified during replacement.
- Redis and Memcached intended network bindings, persistence, replication, and production sizing are not specified.
- PostgreSQL version, schema migration process, backup/restore requirements, and application release process are not specified.
- Self-signed certificates are assumed to be for development only; production certificate requirements need owner confirmation.
- Team coordination should assign separate owners for platform/OS compatibility, web/security, data services/secrets, and application runtime; require peer review for firewall/SSH changes and a shared acceptance checklist before cutover.
