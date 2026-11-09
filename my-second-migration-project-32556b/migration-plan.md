# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant configuration with three local cookbooks: an Nginx multisite/security stack, caching services, and a FastAPI tutorial application backed by PostgreSQL. The migration is moderate in scope because the cookbook count is low, but it includes several imperative shell workarounds, external cookbook dependencies, firewall/SSH hardening, database credentials, and a development-only TLS model. A focused team could complete the first Ansible implementation in approximately 1–2 weeks, with another 1–2 weeks for testing, security remediation, and operational handover. The estimate assumes one target OS is selected and production certificate/database requirements are clarified.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Installs and manages Nginx as a multi-site web server for three configured cluster-local virtual hosts, including site roots, static landing pages, TLS configuration, and service lifecycle.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef
    - Key Features: Three `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local` sites; Nginx configuration templates; enabled-site symlinks; static cookbook files; self-signed RSA certificates; Nginx reload notifications.
    - Security scope: Fail2ban, UFW default-deny firewall rules, SSH root-login/password-authentication hardening, sysctl security settings, and private-key directory permissions.

- **cache**:
    - Description: Installs and enables Memcached and Redis, creates Redis logging storage, and configures a Redis server on port 6379.
    - Path: `cookbooks/cache`
    - Technology: Chef
    - Key Features: External Memcached cookbook, RedisIO cookbook, Redis authentication, log directory creation, and a post-generation configuration cleanup workaround.

- **fastapi-tutorial**:
    - Description: Installs a Python/PostgreSQL runtime, checks out a FastAPI tutorial application, creates a virtual environment, installs its requirements, provisions a PostgreSQL database/user, and runs the application under systemd.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef
    - Key Features: Python 3 and virtualenv, Git checkout from `https://github.com/dibanez/fastapi_tutorial.git` at `main`, PostgreSQL service/database setup, `.env` generation, Uvicorn on `0.0.0.0:8000`, and automatic service restart behavior.

**CRITICAL PATH VERIFICATION:**
All module paths above were present in the supplied repository tree and their primary `recipes/default.rb` entrypoints were reviewed. No Puppet, Salt, or PowerShell modules were identified.

### Infrastructure Files

- `Berksfile`: Chef Supermarket source, local cookbook declarations, and external dependencies (`nginx`, `memcached`, and `redisio`). Replace with Ansible Galaxy collections/roles and explicit role version locking.
- `solo.json`: Chef Solo run list and environment data. Its Nginx site definitions differ from cookbook attribute defaults and should become validated Ansible variables with a single authoritative source.
- `solo.rb`: Chef Solo cache and cookbook-path configuration. Replace with an Ansible inventory, project configuration, and role layout.
- `Vagrantfile`: Provisions `generic/fedora42` using libvirt, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, and forwarded HTTP/HTTPS ports. Retain or replace with an Ansible-compatible Vagrant test harness.
- `vagrant-provision.sh`: Uses `apt-get`, installs Chef/Berkshelf from the network, vendors cookbooks, and runs Chef Solo. Replace with an Ansible installation/bootstrap path; its Debian package assumptions conflict with the Fedora box.
- `cookbooks/nginx-multisite/templates/`: ERB templates for Nginx, security, site, Fail2ban, and sysctl configuration. Convert to Jinja2 templates and preserve only settings confirmed for the target OS.
- `cookbooks/nginx-multisite/files/default/`: Static `index.html` files for CI, status, and test sites. Move to role files or deploy the application content from a versioned artifact.
- `cookbooks/nginx-multisite/attributes/default.rb`: Default site/security/TLS variables. Reconcile with `solo.json`, which currently supplies different document roots.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource exists in the tree but is not referenced by the reviewed default recipes; assess whether it is dead code before migration.

### Target Details

- **Operating System**: The Vagrant target is `generic/fedora42`, but `vagrant-provision.sh` uses `apt-get` and the cookbooks declare Ubuntu >=18.04 and CentOS >=7 support. The intended runtime OS is therefore ambiguous. Select and test a supported platform before implementation; for a new enterprise target, standardize on Red Hat Enterprise Linux 9-compatible systems and use `dnf`, `firewalld`, and platform-correct service/package names. If Fedora 42 is retained for development, add Fedora-specific validation.
- **Virtual Machine Technology**: Vagrant with the libvirt provider; 2 CPUs and 2048 MB memory. The private network is `192.168.121.10`, with host forwards 8080→80 and 8443→443.
- **Cloud Platform**: Not specified. No cloud SDK, cloud-init, or provider metadata was found.

## Migration Approach

Create an Ansible project with an inventory for the Vagrant/test host and separate roles for `nginx_multisite`, `cache`, and `fastapi_tutorial`. Use a site playbook to preserve the current orchestration order while making dependencies explicit. Convert Chef templates to Jinja2, cookbook files to role files, node attributes to group/host variables, and Chef notifications to handlers. Add Molecule or equivalent disposable-VM tests plus an end-to-end Vagrant smoke test covering ports, services, virtual hosts, firewall behavior, Redis authentication, PostgreSQL connectivity, and the FastAPI health path.

### Key Dependencies to Address

- **Chef Infra >=16 / Chef Solo**: Remove Chef runtime and express convergence in Ansible plays, roles, handlers, and idempotent modules.
- **Berkshelf and Chef Supermarket**: Replace with `ansible-galaxy.yml` and pinned collections/roles. Prefer built-in modules where practical; record versions in source control.
- **nginx cookbook ~>12.0**: Replace with an Nginx role or explicit `ansible.builtin.package`, `template`, `file`, and `service` tasks. Validate distro-specific layout and syntax with `nginx -t` before reload.
- **memcached cookbook ~>6.0**: Replace with platform-specific package/service tasks and a constrained listener configuration; do not expose Memcached unintentionally.
- **redisio ~>7.2.4**: Replace with a vetted Redis role or explicit installation/configuration tasks. Preserve authentication and service enablement, but eliminate the brittle post-install file-edit workaround by owning the complete Redis configuration.
- **Python, pip, virtualenv, Git, PostgreSQL, libpq-dev**: Use Ansible package, git, `ansible.builtin.pip`, PostgreSQL modules/collection, and systemd tasks. Pin application dependencies through the application lock/requirements strategy rather than installing an unreviewed moving `main` branch.
- **UFW and Fail2ban**: Map to UFW on Debian-family hosts or `firewalld` on RHEL-family hosts; deploy and validate Fail2ban jails using the target distribution's service names and log paths.
- **OpenSSL/CA certificates**: Use Ansible certificate/key deployment for real certificates. Keep self-signed generation isolated to development only.

### Security Considerations

- **Hardcoded credentials**: `cache/recipes/default.rb` contains the Redis password `redis_secure_password_123`; `fastapi-tutorial/recipes/default.rb` contains the PostgreSQL password `fastapi_password` in SQL commands and the generated `.env`. Remove these values from code and store them in Ansible Vault or an approved secret manager, with rotation during migration.
- **Credential file permissions**: The generated FastAPI `.env` is mode `0644` and owned by root, making the database password broadly readable. Use mode `0600`, a dedicated service account, and preferably systemd environment credentials or a protected secret file.
- **Privilege reduction**: The FastAPI systemd unit runs Uvicorn as `root`. Create a dedicated non-root application user, restrict the working tree, and explicitly set writable directories.
- **TLS certificates and keys**: The current recipe generates self-signed certificates for development, stores keys under `/etc/ssl/private`, and references fixed per-host filenames. Use Vault/secret-manager-backed certificate material or an ACME workflow in non-development environments; protect keys with restrictive ownership/modes and avoid logging secret content.
- **SSH hardening**: `solo.json` enables root-login and password-authentication disabling. Implement with `sshd_config` drop-ins and a validated handler, ensuring an administrative key-based access path exists before applying the change to avoid lockout.
- **Firewall exposure**: Current rules allow SSH, HTTP, and HTTPS but do not address Redis, Memcached, PostgreSQL, or FastAPI access explicitly. Bind internal services to loopback/private interfaces where possible and define a documented least-privilege ingress policy.
- **Nginx and Fail2ban**: Preserve security headers and jail behavior from the reviewed templates, validate configuration before reload, and confirm that log paths match the target OS.
- **Supply chain**: The application is cloned from an external `main` branch and pip dependencies are installed at converge time. Pin a commit/release and use trusted package indexes or hashes for repeatable builds.

### Technical Challenges

- **OS mismatch**: Fedora Vagrant provisioning uses `apt-get`, while cookbook metadata includes Ubuntu/CentOS support and the requested default target may be RHEL-like. Resolve the platform decision first and build OS-specific variables/tasks.
- **Conflicting configuration sources**: Cookbook defaults use `/opt/server/{test,ci,status}`, whereas `solo.json` uses `/var/www/{test,ci,status}`. Establish precedence and migrate the intended paths explicitly.
- **Imperative Chef workarounds**: Redis uses a Ruby block to delete generated directives, and SSH/UFW use shell commands with text substitutions. Replace with fully managed configuration files and native Ansible modules to improve idempotence and auditability.
- **Database idempotence**: The current `psql ... || true` commands hide failures and may not safely model existing users/databases. Use PostgreSQL modules, explicit ownership/privileges, and controlled password handling.
- **TLS lifecycle**: Self-signed certificates are suitable only for the stated development scenario. Production certificate issuance, renewal, SANs, trust distribution, and secret storage are not defined.
- **Service and path portability**: `www-data`, `ssh`, Nginx site directories, PostgreSQL service naming, and UFW are distribution-dependent. Parameterize these and test on the selected OS.
- **Application readiness**: The Chef recipe starts FastAPI but does not show migrations, health checks, dependency locking, or a reverse-proxy relationship to Nginx. Define readiness, restart, logging, and deployment rollback behavior.
- **Testing and coordination**: Preserve the three virtual hosts and forwarded ports in a shared acceptance test. Assign ownership by role, require peer review for firewall/secrets changes, and document a rollback path to the existing Chef/Vagrant flow until parity is demonstrated.

### Migration Order

1. **Foundation and test harness**: Decide the target OS, create Ansible inventory/collections, reproduce the Vagrant/libvirt test host, and establish linting, check mode, idempotence, and service validation.
2. **nginx-multisite**: Migrate the lowest application coupling first, including templates, static files, virtual hosts, TLS development behavior, security controls, and handler-driven reloads. Validate all three sites and firewall/SSH safeguards.
3. **cache**: Migrate Redis and Memcached with vaulted credentials, bounded network exposure, complete managed configuration, and persistence/logging decisions. Test authentication and service restarts.
4. **fastapi-tutorial**: Migrate PostgreSQL, application user/environment, pinned source/dependencies, systemd service, and health checks. This is highest risk because it combines external source, database state, secrets, and runtime behavior.
5. **Cutover and hardening**: Run side-by-side acceptance tests, remediate security findings, document operations, train maintainers, and retire Chef only after rollback and backup procedures are verified.

### Assumptions

- The repository represents a development/test environment rather than a production-ready deployment; the self-signed certificates and cluster-local names support that interpretation.
- `solo.json` is the effective runtime configuration, but the conflicting cookbook attribute paths require owner confirmation.
- The three local cookbooks are the complete migration scope; external cookbook implementations were not present locally and must be replaced or evaluated separately.
- The external Nginx, Memcached, and RedisIO cookbooks may introduce behavior not visible in this repository; parity testing is required.
- The `generic/fedora42` image is intentional for development, although the shell bootstrap is Debian-specific. No final production OS was specified.
- No cloud platform, load balancer, DNS provider, external secret manager, certificate authority, backup policy, monitoring platform, or CI pipeline is defined in the reviewed files.
- PostgreSQL is intended to run locally with a local FastAPI database; replication, backups, migrations, and production data retention are out of scope until confirmed.
- Redis authentication and the PostgreSQL credentials shown are examples but must be treated as compromised secrets and rotated.
- The FastAPI repository's current `main` branch and requirements are trusted; migration should instead pin and review them.
- `resources/lineinfile.rb` is unused by the reviewed entrypoints and may be excluded after confirming no external consumer.
- Team roles should be assigned for platform/OS, web/security, cache, application/database, testing, and operations documentation; each role should own acceptance criteria and rollback notes.
