# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant environment containing three local cookbooks: an Nginx multi-site and host-security stack, a caching stack, and a FastAPI tutorial application with PostgreSQL. The migration is moderate in implementation risk rather than scale: the inventory is small, but the current code contains imperative shell commands, development-only certificates, hardcoded credentials, external cookbook behavior, and an OS mismatch between the Vagrant image and the provisioning script. A two-person team should plan approximately 1–2 weeks for a functional Ansible conversion, including testing and security remediation; production hardening, certificate integration, and deployment integration may extend this to 3–4 weeks.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Installs and configures Nginx as a multi-site web server for three SSL-enabled virtual hosts, including static test, CI, and status content.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Nginx service management, per-site document roots, `sites-available`/`sites-enabled` configuration, templated global and security configuration, removal of the default site, self-signed certificate generation, Fail2ban, UFW, SSH hardening, and sysctl settings.
    - Migration notes: Split into Ansible roles or clearly separated task files for Nginx, TLS, sites/content, firewall, Fail2ban, and SSH hardening. Preserve the site data structure as role defaults or group/host variables, but move environment-specific values out of the cookbook.

- **cache**:
    - Description: Installs and enables Memcached and Redis as local caching services and configures Redis on port 6379 with password authentication.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: Inclusion of the external Memcached cookbook, Redisio installation and enablement, Redis log directory creation, password configuration, and a post-configuration workaround that removes several replication-related settings.
    - Migration notes: Replace cookbook inclusions with explicit Ansible package, configuration-template, service, and handler tasks. Validate the Redis package/version and configuration syntax before carrying forward the imperative cleanup workaround.

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI tutorial application from GitHub, installs Python and PostgreSQL prerequisites, creates a Python virtual environment, installs application requirements, creates a PostgreSQL database/user, and runs the application under systemd.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Git checkout of `https://github.com/dibanez/fastapi_tutorial.git` at `main`, Python virtualenv, PostgreSQL service, database and user creation, `.env` generation, and Uvicorn on `0.0.0.0:8000` under a root-owned systemd service.
    - Migration notes: Use `ansible.builtin.git`, package/service modules, `community.postgresql` database modules, `ansible.builtin.pip` or a virtualenv-aware dependency task, a systemd template, and an application user. Pin the source revision and dependencies for repeatable deployments.

**CRITICAL PATH VERIFICATION:**
Only the three cookbook paths listed above were present in the supplied repository tree. No Puppet modules, Salt states, PowerShell modules, or additional Chef cookbooks were identified.

### Infrastructure Files

- `Berksfile`: Defines the Chef Supermarket source, local cookbook paths, and external `nginx ~> 12.0`, `memcached ~> 6.0`, and `redisio ~> 7.2.4` dependencies. Replace this dependency graph with Ansible roles, collections, and system packages; retain versions in a documented compatibility matrix.
- `solo.json`: Chef Solo run list and environment data. It defines the three sites, document roots, SSL paths, and security switches. Convert these values to Ansible inventory/group variables, with secrets removed from plaintext.
- `solo.rb`: Chef Solo cache, cookbook search paths, and logging configuration. It has no direct Ansible equivalent; replace with project configuration, inventory, and controller logging conventions.
- `Vagrantfile`: Provisions `generic/fedora42` on libvirt with 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, HTTP/HTTPS forwarded ports, and an rsynced repository at `/chef-repo`. Adapt it to install Ansible or use the host/controller to run `ansible-playbook` against the guest.
- `vagrant-provision.sh`: Installs packages, Chef, Berkshelf, cookbook dependencies, and runs Chef Solo. Replace it with an Ansible bootstrap/provisioning path; do not retain the remote `curl | bash` installation pattern without an explicit trust and checksum policy.
- `cookbooks/nginx-multisite/templates/`: Contains Nginx, site, security, Fail2ban, and sysctl templates. Port these to Jinja2 templates and add validation handlers such as `nginx -t` and `sysctl --system`.
- `cookbooks/nginx-multisite/files/`: Contains static `index.html` content for the test, CI, and status sites. Copy these with Ansible and verify their intended production role.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: A custom Chef resource that performs line replacement and timestamped backups. Replace with `ansible.builtin.lineinfile`, `blockinfile`, or a managed template; review regex behavior before migration.

### Target Details

- **Operating System**: The Vagrant target is `generic/fedora42`, but `vagrant-provision.sh` uses `apt-get`, and cookbook metadata supports Ubuntu >=18.04 and CentOS >=7. The actual target OS is therefore ambiguous. Confirm whether Fedora 42 is intentional. For Ansible, use OS-family conditionals or standardize on a supported RHEL/Fedora-family image; do not assume Debian package names or service names.
- **Virtual Machine Technology**: libvirt via Vagrant, with an rsync-synced repository and 2 vCPUs/2 GB RAM.
- **Cloud Platform**: Not specified. The repository is designed for local Vagrant/libvirt testing rather than a named cloud provider.

## Migration Approach

Create an Ansible project with an inventory for the Vagrant guest, a site playbook, and roles such as `nginx_multisite`, `cache`, `fastapi_tutorial`, and optionally `host_security`. Start with check mode and a disposable VM, then test idempotence by running the playbook twice. Use handlers for Nginx, Fail2ban, SSH, Redis, PostgreSQL, and FastAPI service changes. Add Molecule or equivalent converge/verify tests where practical, and keep application, infrastructure, and secret ownership clear between teams.

### Key Dependencies to Address

- **Nginx cookbook `~> 12.0`**: Replace with `ansible.builtin.package`, managed Nginx templates, site symlinks, service handlers, and explicit syntax validation. Confirm Fedora/RHEL versus Debian path differences.
- **Memcached cookbook `~> 6.0`**: Replace with package and service tasks, with a deliberately managed configuration and listener policy.
- **Redisio cookbook `~> 7.2.4`**: Replace with a supported Ansible role or a controlled package/template implementation. Reproduce password protection and service enablement, then eliminate or justify the current replication-setting cleanup.
- **Chef >=16, Chef Solo, Berkshelf**: Remove from the runtime path. Pin `ansible-core` and required collections in `collections/requirements.yml` and document controller/Python requirements.
- **Python 3, pip, venv, Git, PostgreSQL, libpq-dev, OpenSSL, ca-certificates, Fail2ban, UFW**: Map to OS-family-specific package names and services. PostgreSQL database/user operations should use `community.postgresql` and become a database administrative user rather than shelling out to `psql`.
- **FastAPI Git repository and `requirements.txt`**: Pin a commit or release, verify repository ownership, and consider a deployment artifact/image instead of installing unpinned `main` branch dependencies during provisioning.
- **Systemd and Uvicorn**: Use a templated unit, a dedicated least-privilege service account, explicit environment handling, and a restart handler.

### Security Considerations

- **Hardcoded Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Move it to Ansible Vault or an external secret manager, rotate it, and avoid exposing it in task output.
- **Hardcoded PostgreSQL credentials**: `fastapi-tutorial` embeds `fastapi_password` in SQL commands and `/opt/fastapi-tutorial/.env` with mode `0644`. Rotate the credential, use Vault, make the file readable only by the application account, and prevent passwords from appearing in logs.
- **Application privilege**: The FastAPI systemd unit runs as `root`. Create a dedicated application user, restrict directory ownership, and assess whether binding through Nginx is preferable to exposing Uvicorn directly on all interfaces.
- **TLS certificates and private keys**: The Nginx cookbook generates self-signed RSA certificates valid for 365 days under `/etc/ssl`, with private keys group-readable by `ssl-cert`. This is suitable only for development. Define production certificate issuance/renewal, SANs, key permissions, rotation, and Vault/certificate-manager integration.
- **Firewall and SSH hardening**: The configuration enables UFW, allows SSH/HTTP/HTTPS, disables root login, and disables password authentication. Translate these to the target OS firewall module and OpenSSH configuration, verify console/key access before applying, and avoid locking out the migration team.
- **Fail2ban and sysctl**: Preserve the intended jail and kernel settings only after validating them on the target OS. Add tests for valid configuration and service recovery.
- **Supply-chain exposure**: The provisioning script installs Chef via an unpinned remote script, installs Berkshelf dynamically, clones a public GitHub `main` branch, and installs Python requirements. Replace with pinned, checksummed, reviewed dependencies and preferably CI-built artifacts.
- **Credential pattern by module**: `cache` has a Redis password; `fastapi-tutorial` has PostgreSQL credentials and a database URL; `nginx-multisite` references certificate and private-key paths but does not store existing certificate material in the repository. No encrypted data bags or Chef Vault usage were visible in the reviewed files.

### Technical Challenges

- **OS inconsistency**: Fedora 42 is provisioned with `apt-get`, while cookbook metadata targets Ubuntu/CentOS. Establish the supported OS first and add package/service/path abstractions before testing.
- **Chef dependency behavior**: External cookbook recipes may provide defaults and platform logic not visible in the local files. Inventory the resolved Berkshelf cookbook versions and compare effective behavior before replacing them.
- **Imperative configuration hacks**: Redis uses a Ruby block to edit generated configuration, and the custom `lineinfile` resource uses dynamic regular expressions and timestamp backups. Replace with declarative templates/modules and verify idempotence.
- **Template parity**: The reviewed recipes reference Nginx, security, Fail2ban, sysctl, and site templates. Porting requires preserving all directives and testing `nginx -t`, while accounting for distribution-specific Nginx layouts.
- **Database idempotence**: Shell commands suppress errors with `|| true`, which can hide partial or incorrect database state. Use PostgreSQL modules with explicit ownership, privileges, and migration-safe checks.
- **Application reproducibility**: Tracking `main` and installing the current requirements at provisioning time can produce drift. Pin code/dependencies and define upgrade/rollback behavior.
- **Certificate and DNS assumptions**: The three hostnames, certificate filenames, and local hosts-file examples are development-oriented. Confirm DNS, certificate authority, and external load-balancer responsibilities.
- **Firewall ordering and connectivity**: Enabling a default-deny firewall or changing SSH authentication can interrupt provisioning. Apply access rules and validation before enforcement, with console recovery available.

### Migration Order

1. **Foundation and test harness**: Confirm the target OS, update Vagrant provisioning for Ansible, create inventory/group variables, pin Ansible collections, and establish lint, syntax, check-mode, and idempotence tests.
2. **nginx-multisite static baseline**: Migrate directories, static files, Nginx installation, the three virtual hosts, service handlers, and configuration validation without initially enforcing disruptive SSH/firewall changes.
3. **cache**: Migrate Memcached and Redis with secure secret handling and explicit configuration; test service startup, authentication, and repeatability.
4. **fastapi-tutorial**: Deploy the application with pinned source/dependencies, PostgreSQL modules, a dedicated service account, protected environment data, and systemd verification.
5. **host security and production TLS**: Enable SSH hardening, UFW/firewall, Fail2ban, sysctl controls, and trusted certificate issuance after access and rollback procedures are tested.
6. **Cutover and decommissioning**: Compare Chef and Ansible state, run parallel validation where possible, document ownership/runbooks, then remove Chef/Berkshelf bootstrap dependencies.

### Assumptions

- The three local cookbook directories are the complete module inventory for this repository.
- `solo.json` is the intended effective Chef run list and supplies the primary environment overrides.
- The three sites and static HTML files are primarily a development/test workload; production traffic, DNS, and certificate issuance are not represented.
- The Vagrant Fedora image is intentional, although the current bootstrap script is Debian-oriented; this must be resolved before implementation.
- No production inventory, cloud integration, external secret manager, certificate store, database backup policy, monitoring, or CI/CD pipeline was present in the reviewed files.
- External cookbook implementations are not vendored in the repository; their resolved versions and behavior must be captured before final parity sign-off.
- The FastAPI repository, its branch, and its dependency file are trusted and available to the target host; migration should validate and pin them rather than relying on that assumption.
- PostgreSQL is intended to run locally on the same host as FastAPI, and Redis/Memcached are intended to be local services; confirm topology before designing inventories.
- The current self-signed certificate behavior is acceptable only for local testing and must not be promoted unchanged.
- Application database credentials and Redis credentials are examples that require rotation, not values to preserve.
- A migration team will own separate workstreams for platform/OS, web/security, caching, application/database, secrets, and test/cutover coordination.
- Estimated timing assumes access to a disposable Vagrant/libvirt test environment and a decision on the supported OS during the first planning phase.
