# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository is a Chef Solo repository containing exactly three local Chef cookbooks. The migration scope is small in module count but requires security remediation and OS clarification. The main risks are plaintext Redis/PostgreSQL credentials, self-signed TLS generation, imperative configuration edits, a mutable application Git branch, and an inconsistency between the Fedora Vagrant image and the `apt-get`-based provisioning script. Estimated effort is 1–2 weeks for implementation and 2–3 weeks including testing, security review, documentation, and handover.

## Module Migration Plan

The source technology is **Chef**: cookbook directories contain `metadata.rb` and `recipes/default.rb`; `Berksfile`, `solo.rb`, `solo.json`, and `chef-solo` provisioning confirm the Chef Solo execution model.

### MODULE INVENTORY

The following three modules are separate local cookbooks and each has a confirmed path in the repository tree and a reviewed `recipes/default.rb` entrypoint.

#### 1. `cache`

- **Path:** `cookbooks/cache`
- **Technology:** Chef cookbook
- **Entrypoint:** `cookbooks/cache/recipes/default.rb`
- **Description:** Installs and includes Memcached and Redis, configures a Redis server on port 6379, enables Redis, and creates `/var/log/redis`.
- **Key Features:** Redis password authentication, Redis server configuration through the `redisio` cookbook, Memcached inclusion, Redis log-directory ownership/mode, and a Ruby configuration cleanup block removing replication-related directives.
- **Migration target:** An Ansible cache role using OS packages or a pinned Redis role, explicit Redis configuration templates, service handlers, restricted network binding, and Ansible Vault for credentials.

#### 3. `nginx-multisite`

- **Path:** `cookbooks/nginx-multisite`
- **Technology:** Chef cookbook
- **Entrypoint:** `cookbooks/nginx-multisite/recipes/default.rb`
- **Description:** Orchestrates Nginx, host security, SSL, and multiple site configuration recipes for `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`.
- **Key Features:** Nginx installation/service management, per-site document roots and virtual hosts, static site files, self-signed certificates, Fail2ban, UFW default-deny rules with SSH/HTTP/HTTPS exceptions, sysctl security settings, disabled root SSH login, and disabled SSH password authentication.
- **Related source:** `attributes/default.rb`, `recipes/nginx.rb`, `recipes/security.rb`, `recipes/sites.rb`, `recipes/ssl.rb`, ERB templates, and static files under this cookbook.
- **Migration target:** An Ansible web/security role split into Nginx, sites, TLS, and hardening task files with Jinja2 templates, handlers, OS-family firewall variables, and validated configuration reloads.

**Inventory completeness:** No Puppet `manifests/init.pp` files or PowerShell `.psd1` module manifests are present in the supplied repository tree. No additional Chef cookbook directories are shown beyond the three entries above.

### Infrastructure Files

- `Berksfile`: Chef Supermarket source, three local cookbook declarations, and external dependencies: `nginx ~> 12.0`, `memcached ~> 6.0`, and `redisio ~> 7.2.4`. Replace Berkshelf with Ansible Galaxy requirements and pinned role/collection versions.
- `solo.json`: Chef Solo run list and node overrides. It invokes `nginx-multisite`, `cache`, and `fastapi-tutorial`; migrate its site, SSL, and security variables to inventory/group vars and Vault.
- `solo.rb`: Chef Solo cache path, cookbook paths, logging, and output settings. Replace with Ansible project layout and controller configuration.
- `Vagrantfile`: Uses `generic/fedora42`, libvirt, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, and forwarded ports 80/443. Update provisioning to Ansible or use it as a test harness.
- `vagrant-provision.sh`: Installs Chef and Berkshelf, vendors dependencies, and runs Chef Solo. Replace with an Ansible installation/invocation workflow.
- `cookbooks/nginx-multisite/attributes/default.rb`: Defines site roots, certificate paths, and security defaults; convert to role defaults and environment variables.
- `cookbooks/nginx-multisite/templates/`: ERB templates for Nginx, virtual sites, security, Fail2ban, and sysctl; convert to Jinja2.
- `cookbooks/nginx-multisite/files/`: Static HTML content for the three sites; migrate with `copy` or an artifact deployment task.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource; confirm consumers and replace with Ansible `lineinfile`, `blockinfile`, or managed templates.

### Target Details

- **Operating System:** Ambiguous. `Vagrantfile` specifies Fedora 42, while `vagrant-provision.sh` uses `apt-get` and the cookbooks declare Ubuntu >=18.04/CentOS >=7 support. Select Fedora-compatible `dnf` tasks or change the test image to Ubuntu. Do not finalize a production OS until confirmed; RHEL 9 is only a default planning target if stakeholders approve it.
- **Virtual Machine Technology:** Vagrant using the libvirt provider.
- **Cloud Platform:** Not specified; no AWS, Azure, or GCP configuration is present.

## Migration Approach

### Key Dependencies to Address

- `nginx ~> 12.0`: Replace the external Chef cookbook with an internal Ansible Nginx role, distribution packages, templates, and handlers.
- `memcached ~> 6.0`: Replace with OS package/service tasks and explicit bind/firewall policy.
- `redisio ~> 7.2.4`: Replace with a pinned Redis implementation and managed config; eliminate the Ruby post-processing hack.
- PostgreSQL packages and commands: Use `community.postgresql.postgresql_db` and `postgresql_user` with idempotent behavior.
- Git/Python dependencies: Use Ansible Git and virtualenv/pip tasks; pin the application commit and dependency versions instead of `main` and unconstrained requirements.
- systemd: Template the FastAPI service, use `systemd_service`, and notify daemon reload handlers.
- Chef/Berkshelf: Remove Chef installation, Berkshelf vendor steps, and cookbook paths; add pinned Ansible core/collection requirements.

### Security Considerations

- Redis password `redis_secure_password_123` is hardcoded in `cookbooks/cache/recipes/default.rb`. Treat it as exposed, rotate it, and store the replacement in Ansible Vault or an external secret manager.
- PostgreSQL password `fastapi_password` is embedded in shell commands and `/opt/fastapi-tutorial/.env` with mode `0644`. Rotate it, avoid shell-visible secrets, and write the environment file with restrictive permissions.
- The FastAPI service runs as `root`; create a dedicated non-root account and restrict application/database access.
- SSL recipes generate one-year self-signed keys under `/etc/ssl/private`. Keep this only for development and define a production certificate issuance/renewal process with controlled private-key access.
- Replace SSH `sed` edits with validated Ansible configuration and ensure a working key-based administrative account before disabling root/password login.
- Reproduce UFW/Fail2ban/sysctl controls using OS-appropriate modules. Keep port 8000 and Redis 6379 private unless explicitly required.
- Pin installer, package, Git, and Python dependency versions; the current `curl | bash` Chef installation and mutable public Git branch are supply-chain risks.

### Technical Challenges

- Resolve Fedora versus Debian/Ubuntu assumptions before role implementation; package names, firewall tooling, service names, and users differ.
- Reconcile Nginx document-root defaults in `attributes/default.rb` (`/opt/server/...`) with `solo.json` overrides (`/var/www/...`).
- Convert Chef notifications, templates, links, and the custom resource into idempotent Ansible tasks and handlers.
- Replace PostgreSQL `|| true` commands with explicit database modules and proper failure reporting.
- Validate the external application repository actually contains `requirements.txt` and `app.main`; these are assumptions in the cookbook.
- Establish local DNS or `/etc/hosts` entries for the `.cluster.local` names; the Vagrant host-entry provisioner is commented out.
- Test Nginx syntax, TLS permissions, SSH lockout prevention, firewall rules, Redis authentication, database access, and second-run idempotence.

### Migration Order

1. Confirm target OS, inventory structure, Ansible version, collection versions, secret backend, and Vagrant test strategy.
2. Migrate `nginx-multisite` because it establishes the web, TLS, firewall, SSH, Fail2ban, and sysctl baseline.
3. Migrate `cache` with rotated Vault-backed credentials and restricted Redis/Memcached exposure.
4. Migrate `fastapi-tutorial`, including PostgreSQL, pinned application deployment, secret-backed `.env`, and non-root systemd service.
5. Run integration/idempotence/security tests, document operations and rollback, obtain application/infrastructure/security sign-off, then retire Chef.

Estimated effort: 1–2 days for foundation, 2–3 days for `nginx-multisite`, 1–2 days for `cache`, 2–3 days for `fastapi-tutorial`, and 2–4 days for testing, remediation, documentation, and handover.

### Assumptions

- The three confirmed cookbook paths are the complete local module inventory.
- Chef Solo is the only source automation technology in scope.
- `solo.json` represents the intended combined deployment, although it overrides some cookbook defaults.
- The Vagrant machine is a development environment and does not establish the production OS.
- The external FastAPI repository, its `main` branch, `requirements.txt`, and `app.main` remain available but were not audited here.
- The visible Redis and PostgreSQL credentials must be treated as compromised.
- The commented `ssl_certificate` Berkshelf dependency is inactive.
- Production DNS, certificates, secret storage, monitoring, backup, compliance, availability, and rollback requirements are unspecified.
- Coordination requires an infrastructure owner for OS/network/security decisions, an application owner for FastAPI/PostgreSQL validation, and a security reviewer for credential rotation, SSH/firewall policy, and TLS. Role variables, rendered configurations, test evidence, and operating runbooks should be reviewed before Chef is removed.

This is useless comment.