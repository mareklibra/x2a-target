# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant configuration with three local cookbooks: an Nginx multi-site/security stack, caching services, and a FastAPI tutorial application backed by PostgreSQL. The target Ansible implementation should preserve the current single-host development workflow first, then make environment-specific values, certificates, credentials, firewall policy, and application deployment explicit. The migration is moderate rather than large: approximately 3–5 working days for a functional conversion and 1–2 additional weeks for testing, security remediation, production hardening, and team handover, assuming the FastAPI application's operational requirements are understood.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server configuration for three SSL-enabled virtual hosts (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), including static content, HTTP-to-HTTPS redirects, security headers, rate limiting, compression, and per-site logs.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Nginx service management, templated `nginx.conf` and virtual hosts, self-signed certificate generation, Fail2ban, UFW, SSH hardening, sysctl network hardening, and static site files.
    - Ansible decomposition: Separate roles or task includes for Nginx, virtual hosts/content, TLS, host security, firewall, Fail2ban, and sysctl settings. Preserve handler-driven reload/restart behavior.

- **cache**:
    - Description: Installs and enables Memcached and Redis, creates Redis logging storage, and configures a Redis instance on port 6379 with password authentication.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: External `memcached` and `redisio` cookbook integration, Redis password configuration, Redis enablement, and a post-configuration workaround that removes selected replication directives.
    - Ansible decomposition: A cache role using distribution packages or supported Ansible roles, with explicit Redis configuration and service handlers. Replace the Chef `ruby_block` workaround with a managed configuration template or version-specific task only if still required.

- **fastapi-tutorial**:
    - Description: Deploys the FastAPI tutorial application from the `dibanez/fastapi_tutorial` Git repository, creates a Python virtual environment, installs requirements, configures PostgreSQL and a database/user, writes an application `.env`, and runs Uvicorn as a systemd service.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Python 3 tooling, Git checkout of the `main` branch, virtualenv, PostgreSQL packages/service, database initialization, systemd unit, and Uvicorn listening on all interfaces at port 8000.
    - Ansible decomposition: Application deployment role plus PostgreSQL/database role or a clearly defined external database dependency. Use `ansible.builtin.git`, package/virtualenv modules, `community.postgresql`, `ansible.builtin.systemd`, templates, and handlers. Pin the application revision and Python dependencies before production use.

**CRITICAL PATH VERIFICATION:**
All module paths above were present in the supplied repository tree and their primary `recipes/default.rb` and metadata files were reviewed. No Puppet, Salt, or PowerShell modules were found.

### Infrastructure Files

- `Berksfile`: Defines local cookbook paths and external Supermarket dependencies: `nginx ~> 12.0`, `memcached ~> 6.0`, and `redisio ~> 7.2.4`; a commented `ssl_certificate` dependency is not active. Replace these cookbook dependencies with Ansible roles, collections, native modules, or maintained packages.
- `Vagrantfile`: Defines the development target as `generic/fedora42`, hostname `chef-nginx`, a libvirt VM with 2 GB RAM and 2 CPUs, private IP `192.168.121.10`, and HTTP/HTTPS forwarded ports. It rsyncs the repository to `/chef-repo` and invokes the shell provisioning script.
- `solo.json`: Chef Solo run list and node attributes. It supplies the three cookbooks, site definitions, SSL directories, and security flags. Its site document roots conflict with the cookbook defaults: `/var/www/...` here versus `/opt/server/...` in `attributes/default.rb`; resolve this explicitly during migration.
- `solo.rb`: Chef Solo cache, cookbook search paths, logging level, and stdout logging. Its `/chef-repo` paths are development-only and should not be copied into the Ansible layout.
- `vagrant-provision.sh`: Installs packages, Chef, Berkshelf, vendors dependencies, accepts the Chef license, and runs Chef Solo. It uses `apt-get` even though the Vagrant box is Fedora 42, so the current provisioning is internally inconsistent. Replace it with an Ansible provisioner or a distribution-aware bootstrap.
- `cookbooks/nginx-multisite/templates/default/*`: Nginx, virtual-host, security, Fail2ban, and sysctl source templates. These are the primary configuration inputs to preserve or modernize as Jinja2 templates.
- `cookbooks/nginx-multisite/files/default/*`: Static `index.html` content for test, CI, and status sites. Migrate as role files or application content artifacts.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom resource exists in the tree but was not referenced by the reviewed default recipes; assess whether it is dead code before migration.

### Target Details

- **Operating System**: Development configuration explicitly selects Fedora 42, but `vagrant-provision.sh` uses Debian/Ubuntu `apt-get`, and cookbook metadata supports Ubuntu >=18.04 and CentOS >=7. The target OS is therefore unresolved. Decide on Fedora/RHEL-compatible packaging or change the development box to Ubuntu before implementation. Ansible should use OS-family variables rather than assume `apt`.
- **Virtual Machine Technology**: Vagrant with the libvirt provider; 2 vCPUs and 2 GB RAM. The private development address is `192.168.121.10`, with host forwards 8080→80 and 8443→443.
- **Cloud Platform**: Not specified. No cloud provider SDK, metadata endpoint, or cloud-init configuration was found.

## Migration Approach

### Key Dependencies to Address

- **Chef Solo/Berkshelf**: Replace the Chef run list with an Ansible playbook and roles, for example `site.yml` calling `nginx_multisite`, `cache`, and `fastapi_tutorial` roles. Use `ansible-galaxy` requirements with pinned collection/role versions.
- **`nginx` cookbook `~> 12.0`**: Use native Ansible package, template, file, service, and handler tasks, or a reviewed maintained Nginx role. Preserve enabled-site links and validate with `nginx -t` before reload.
- **`memcached` cookbook `~> 6.0`**: Install the OS package and manage its service/configuration using native modules or a maintained role. Confirm the required bind address and access policy; the source does not specify them.
- **`redisio` cookbook `~> 7.2.4`**: Replace with distribution Redis packages or a maintained role. Model port, password, logging, persistence, and service enablement as variables. Verify the source workaround against the selected Redis version.
- **PostgreSQL packages and client tools**: Use OS-family package variables and `community.postgresql` for idempotent database/user creation. Ensure the PostgreSQL Python driver is available to the Ansible execution context.
- **Python/Git/Uvicorn dependencies**: Install Python tooling, create a virtual environment, deploy a pinned Git revision, and run a controlled `pip install` from the application's requirements file. Consider an artifact or immutable deployment rather than cloning `main` at provisioning time.
- **systemd, UFW, Fail2ban, OpenSSL, and sysctl**: Map directly to Ansible modules and handlers, with distribution-specific service names (`ssh` versus `sshd`) and firewall support tested on the selected OS.

### Security Considerations

- **Hardcoded Redis credential**: `redis_secure_password_123` is embedded in `cookbooks/cache/recipes/default.rb`. Treat it as compromised, rotate it, and store the replacement in Ansible Vault or an external secret manager. Do not place it in defaults, Git, command-line arguments, or world-readable configs.
- **Hardcoded PostgreSQL credentials**: The FastAPI recipe creates user `fastapi` with password `fastapi_password` and writes the same password into `/opt/fastapi-tutorial/.env` with mode `0644`. Replace with Vault-managed credentials, use a restrictive mode such as `0600`, and avoid shell commands containing secrets.
- **TLS certificates and keys**: The Nginx cookbook generates self-signed RSA 2048 certificates for each site under `/etc/ssl/certs` and private keys under `/etc/ssl/private`, with key mode 0640 and `ssl-cert` group ownership. These are development certificates only. For real environments, provision certificates through a controlled PKI/ACME process, protect private keys with Vault or a certificate-management system, and define renewal/rotation.
- **TLS configuration**: The templates restrict protocols to TLS 1.2/1.3, define explicit ciphers, enable HSTS, and set security headers. Validate current cipher guidance and avoid enabling HSTS before HTTPS is reliable for every hostname.
- **Firewall and SSH**: UFW defaults to deny and allows SSH/HTTP/HTTPS; SSH root login and password authentication are disabled. Confirm administrative access and an approved key-based user before applying these tasks to avoid lockout. The source does not explicitly allow application port 8000, Redis, or Memcached, which is appropriate if they remain local-only but must be verified.
- **Nginx hardening**: The configuration includes request limits, timeout limits, hidden-file blocking, rate-limit zones, security headers, and Fail2ban jails. Test the very small `client_max_body_size` (1K) against application needs and verify Fail2ban log paths and filters on the target OS.
- **Application privilege**: The systemd service runs FastAPI as `root`. Create a dedicated non-privileged service account, restrict filesystem permissions, and bind to a local/private interface unless external exposure is intentional.
- **Supply chain**: The application is cloned from an unpinned `main` branch and pip requirements are not reviewed here. Pin Git commits and dependency hashes, use trusted repositories, and scan dependencies.
- **Sysctl impact**: IPv6 is disabled and ICMP echo is blocked. Confirm these are intentional for the target network and document operational consequences before applying globally.

### Technical Challenges

- **OS inconsistency**: Fedora 42 is selected while bootstrap commands are Debian-specific and cookbook paths/templates assume Debian conventions such as `www-data`, UFW, `/etc/nginx/sites-enabled`, and `/var/log/auth.log`. Choose and test the supported OS first; introduce variables for package names, users, paths, firewall, and SSH service.
- **Configuration precedence mismatch**: `solo.json` overrides site roots to `/var/www/...`, while cookbook defaults use `/opt/server/...`; the static files are selected by site name. Define one authoritative variable model and verify rendered Nginx roots and content.
- **Certificate lifecycle**: Current certificates are generated imperatively and are not renewed. Build a clear development self-signed mode and a separate production certificate mode, with idempotent renewal and safe Nginx reloads.
- **Redis workaround and idempotence**: The Chef `ruby_block` edits generated Redis configuration by deleting directives. Replace this with a supported template/drop-in and test against the selected Redis release rather than reproducing text manipulation.
- **Database initialization**: The current `psql` commands use `|| true`, masking failures and making state unclear. Use idempotent database modules, explicit ownership, privileges, migrations, and readiness checks.
- **Application readiness and deployment**: Installing dependencies and immediately starting Uvicorn does not prove the application is healthy. Add service ordering, health checks, rollback expectations, and a pinned deployment artifact or revision.
- **Nginx template correctness**: The reviewed site template contains an HTTP server block that is conditionally closed and then opens an HTTPS block; validate rendered syntax for both SSL-enabled and non-SSL cases. Run `nginx -t` before every reload.
- **Testing environment drift**: The Vagrant script advertises addresses that differ from the configured private IP in places, and host-name resolution is commented out. Make test DNS/hosts entries and endpoint checks explicit.

### Migration Order

1. **Foundation and target OS decision**: Establish the Ansible project layout, inventory, variables, Vault strategy, Molecule/Vagrant test target, and supported OS. Resolve Fedora versus Ubuntu and the document-root discrepancy before porting behavior.
2. **`nginx-multisite` baseline**: Migrate packages, directories, static files, Nginx templates, virtual hosts, handlers, and basic service validation. This is the externally visible core and provides an early smoke-test target.
3. **Host security portion of `nginx-multisite`**: Add SSH hardening, firewall, sysctl, Fail2ban, and TLS controls after access recovery has been tested. Roll out firewall/SSH changes cautiously with an emergency access procedure.
4. **`cache`**: Deploy Memcached and Redis after secrets and bind/access decisions are finalized. Test local connectivity, authentication, persistence expectations, and the Redis configuration workaround.
5. **`fastapi-tutorial`**: Migrate PostgreSQL, application checkout, virtualenv, secret file, and systemd service. Replace root execution and hardcoded credentials before considering the role production-ready.
6. **Integration and handover**: Test all three hostnames, HTTP redirects, certificates, Nginx reloads, Fail2ban behavior, database connectivity, Redis authentication, service restarts, and repeatability. Document ownership, variables, rollback, and operational runbooks.

### Assumptions

- The desired first target is the existing Vagrant/libvirt development VM, not a cloud deployment.
- The repository contains exactly three local Chef cookbooks; no additional cookbook code or external cookbook source was supplied beyond the Berksfile constraints.
- `fastapi_tutorial` remains the application source, but its internal code, migrations, supported Python versions, and requirements were not reviewed.
- The three site names and SSL-enabled behavior are intended to remain, although DNS/hosts management is currently incomplete.
- Self-signed certificates are acceptable only for development; production certificate ownership and issuance are not specified.
- PostgreSQL is intended to run on the same host as the FastAPI application, as implied by `localhost` and the local service recipe.
- Redis and Memcached are intended to be local services; no remote clients or replication topology is specified.
- The source's Fedora 42 box may be an accidental change from an Ubuntu-oriented implementation. This must be confirmed before role tasks are finalized.
- The source paths, package names, service names, and users may require adaptation for the selected distribution.
- The `lineinfile` custom resource is unused by the reviewed entrypoints and may be removable, subject to a repository-wide usage check.
- No encrypted Chef data bags, Chef Vault usage, external secret manager, certificate files, or environment-specific inventories were found in the reviewed files; visible credentials are plaintext and require immediate remediation.
- The migration team should assign owners for platform/OS compatibility, Nginx/security, data services, application deployment, and test/operations sign-off. Changes should be reviewed through pull requests and validated in an isolated Vagrant environment before any shared or production rollout.
