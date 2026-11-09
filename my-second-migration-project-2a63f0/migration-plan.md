# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant environment containing three local cookbooks: an Nginx multi-site/security stack, caching services, and a FastAPI tutorial application with PostgreSQL. The migration is moderate in operational risk rather than size: the cookbook count is low, but the implementation contains embedded credentials, imperative shell commands, generated self-signed certificates, service ordering, and an OS/platform mismatch that must be resolved before production use. A reasonable estimate is 2–4 weeks for one engineer, including an initial Ansible conversion, Molecule or VM validation, security remediation, and cutover documentation; parallel work by a platform and application engineer can reduce elapsed time to roughly 1–2 weeks.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx reverse proxy/web server configuration for three SSL-enabled virtual hosts (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), with static site directories and example index pages.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Nginx installation and service management, per-site configuration and symlinks, configurable document roots, self-signed certificate generation, Fail2ban, UFW firewall rules, SSH hardening, sysctl security settings, and templated Nginx/security configuration.
    - Ansible target: A role for Nginx/sites and a separable hardening role; use `ansible.builtin.template`, `package`, `service`, `file`, `copy`, `community.general.ufw` or an OS-appropriate firewall solution, and `ansible.posix.sysctl`.

- **cache**:
    - Description: Installs and enables Memcached and Redis, creates the Redis log directory, configures a Redis listener on port 6379 with authentication, and applies a post-install configuration workaround.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: Memcached dependency recipe, Redisio-managed server configuration, Redis password, Redis service enablement, and an imperative cleanup of replication-related Redis directives.
    - Ansible target: A cache role with explicit Redis and Memcached variables, package/service handlers, a managed Redis configuration template, and a controlled migration of the current workaround. Do not carry the password into plaintext defaults.

- **fastapi-tutorial**:
    - Description: Provisions a FastAPI tutorial application from the `dibanez/fastapi_tutorial` Git repository, installs Python/PostgreSQL operating-system packages, creates a Python virtual environment, installs application requirements, creates a PostgreSQL database/user, and runs Uvicorn through systemd.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Git checkout at `main`, Python venv and pip installation, PostgreSQL service, `fastapi_db` database and `fastapi` user, generated `.env`, root-owned systemd service, and Uvicorn on `0.0.0.0:8000`.
    - Ansible target: An application role or playbook using `ansible.builtin.git`, package/venv tasks, `community.postgresql.postgresql_user` and `postgresql_db`, a templated systemd unit, and handlers for daemon reload/restart. Pin application source and Python dependencies for repeatable deployments.

**CRITICAL PATH VERIFICATION:**
The three cookbook paths above are present in the supplied repository tree, and each module's `metadata.rb` and `recipes/default.rb` was reviewed. No Puppet, Salt, or PowerShell modules were identified.

### Infrastructure Files

- `Berksfile`: Chef Supermarket source, local cookbook declarations, and external cookbook constraints. It is the primary dependency inventory to replace with Ansible collections/roles and explicit package/service tasks.
- `solo.rb`: Chef Solo cache, cookbook search paths, logging, and local execution configuration. Replace with Ansible inventory, `ansible.cfg`, playbooks, and role paths.
- `solo.json`: Chef Solo run list and node attributes. Convert site definitions, SSL paths, and security flags into group/host variables, preferably encrypted where sensitive.
- `Vagrantfile`: Defines a `generic/fedora42` libvirt VM, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, forwarded HTTP/HTTPS ports, and an rsync-mounted repository. It is the likely local integration target, but its network and OS assumptions need correction/verification.
- `vagrant-provision.sh`: Installs Chef using an external installer, installs Berkshelf, vendors dependencies, and executes Chef Solo. Replace with an Ansible provisioner or `ansible_local`/host Ansible workflow; remove the Chef bootstrap path after validation.
- `cookbooks/nginx-multisite/templates/`: Contains the Nginx, site, security, Fail2ban, and sysctl templates that must be ported to Jinja2 after reviewing their directives for target-OS compatibility.
- `cookbooks/nginx-multisite/files/default/`: Static `index.html` files for the test, CI, and status sites; move to role `files/` and deploy with site-specific variables.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource, requiring review during implementation to determine whether it is used by the reviewed recipes and whether it maps directly to `ansible.builtin.lineinfile`.

### Target Details

- **Operating System**: The Vagrant box is explicitly `generic/fedora42`, but `vagrant-provision.sh` uses `apt-get`, and cookbook metadata supports Ubuntu >=18.04 and CentOS >=7. The source therefore has an unresolved OS mismatch. Select and test one supported target—likely Fedora/RHEL-family if retaining the Vagrant box, or Ubuntu if the apt-based bootstrap is authoritative. Do not assume Debian package names, service names, firewall tooling, or Nginx paths until decided.
- **Virtual Machine Technology**: Vagrant with the `libvirt` provider; 2 CPUs and 2048 MB memory. The exact host/libvirt availability is environment-dependent.
- **Cloud Platform**: Not specified. No cloud provider integration is present.

## Migration Approach

### Key Dependencies to Address

- **Chef >= 16 / Chef Solo**: Replace the Chef Solo run list with an Ansible inventory and one or more idempotent plays. Preserve role boundaries rather than reproducing Chef's global node attributes.
- **Berkshelf and Chef Supermarket**: Remove Berkshelf/vendor behavior. The local cookbooks are the only directly maintained modules; replace external cookbooks with Ansible tasks/roles and collections.
- **`nginx` Chef cookbook `~> 12.0`**: Replace with an internally managed Nginx role or a vetted Ansible Galaxy role. Port and validate the repository's custom templates and virtual-host behavior.
- **`memcached` Chef cookbook `~> 6.0`**: Replace with distribution packages, service management, and an explicit Memcached configuration role.
- **`redisio` Chef cookbook `~> 7.2.4`**: Replace with a supported Redis package/role and an explicit configuration template. Confirm Redis version and configuration directive compatibility before retaining the replication-directive workaround.
- **PostgreSQL and Python runtime packages**: Use OS-specific package variables and Ansible PostgreSQL modules. Install `python3`, venv tooling, compiler/client libraries, and PostgreSQL client/server packages according to the selected OS.
- **UFW, Fail2ban, OpenSSL, systemd, and sysctl**: Map these to Ansible modules and handlers. On Fedora/RHEL, use firewalld unless UFW is deliberately installed and supported; use `ansible.posix.firewalld` as appropriate.
- **GitHub application source and pip requirements**: Pin the Git revision rather than tracking `main`, and validate or lock `requirements.txt`. Consider a deployment artifact or image for production rather than installing unreviewed dependencies at convergence time.

### Security Considerations

- **Hardcoded Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Move it to Ansible Vault or an external secret manager, rotate it, and avoid exposing it in task output or generated files.
- **Hardcoded PostgreSQL credential**: `fastapi-tutorial` embeds `fastapi_password` in SQL and in the `.env` database URL. Rotate it, store it in Vault, use PostgreSQL Ansible modules rather than shell SQL, and set restrictive ownership/mode on the environment file (not the current broadly readable `0644`).
- **TLS certificates and private keys**: `nginx-multisite` creates self-signed certificates and private keys under `/etc/ssl/private` with `root:ssl-cert` ownership and mode 0640. Treat keys as secrets, provision trusted certificates through a certificate-management process, and do not generate self-signed certificates for production. Validate hostname/SAN requirements and certificate renewal.
- **SSH hardening**: The source disables root login and password authentication using `sed`. Port this to validated Ansible SSH configuration with syntax checks, a safe handler, and an out-of-band recovery path so access is not lost. Confirm an administrative key/user exists before applying it.
- **Firewall exposure**: The source enables default-deny UFW and permits SSH, HTTP, and HTTPS. Reconcile this with the application listening on port 8000 and the intended Nginx proxy path; do not expose 8000 unless required. Define allowed source networks for SSH rather than allowing it universally.
- **File ownership and service identity**: The FastAPI service runs as `root`, and the application directory is root-owned. Create a dedicated least-privilege service account and define writable directories explicitly.
- **Supply chain and bootstrap**: The provisioning script downloads and executes Chef's installer via `curl`, and the application clones an unpinned `main` branch. Replace with verified package repositories, pinned versions/revisions, checksums/signatures where practical, and CI scanning.
- **Security configuration validation**: Review the unexamined templates for headers, TLS protocol/cipher settings, Fail2ban jails, and sysctl values before porting. Use Ansible check mode, Nginx config tests, SSH validation, and firewall tests in CI.

### Technical Challenges

- **OS inconsistency**: Fedora 42 is declared while provisioning calls `apt-get`, and cookbook metadata also supports Ubuntu/CentOS. Resolve the platform decision first; create OS-specific variables and integration tests.
- **Chef attribute precedence/configuration drift**: `solo.json` overrides Nginx document roots from the cookbook defaults. Preserve the effective desired values deliberately in inventory/group vars and document precedence.
- **Imperative workarounds**: Redis uses a Ruby block to edit generated configuration, and firewall/SSH changes use shell commands. Replace these with fully managed templates/modules and handlers; test idempotence and upgrade behavior.
- **TLS generation and lifecycle**: The current certificate task only checks file existence and generates one-year self-signed certificates. Define certificate ownership, renewal, trust, SANs, and development versus production behavior.
- **Database initialization idempotence**: Shell SQL uses `|| true`, masking errors and making state ambiguous. Use PostgreSQL modules, explicit privileges, and migration-safe checks.
- **Service ordering and application readiness**: The app depends on PostgreSQL and its venv/requirements. Model handlers and systemd dependencies explicitly, and add health checks for Nginx, Redis, PostgreSQL, and FastAPI.
- **Template and path portability**: Nginx site paths, user/group names (`www-data`, `ssl-cert`), service names, and firewall implementations vary by OS. Validate each on the selected distribution.
- **Repository scope versus production scope**: The FastAPI repository and requirements are external to this tree, so application behavior, migrations, and runtime dependencies cannot be completely assessed from this source alone.

### Migration Order

1. **Foundation and platform decision**: Choose Ubuntu or Fedora/RHEL, correct Vagrant provisioning, establish Ansible inventory, Vault policy, CI linting, Molecule/VM tests, and baseline OS facts.
2. **nginx-multisite**: Port static files, virtual hosts, Nginx templates, service handlers, and development TLS first. Add hardening/firewall as a separately reviewed role and verify that access remains available.
3. **cache**: Implement Memcached and Redis with vaulted credentials, explicit configuration, version testing, and service health checks. Validate clients and Redis behavior before removing Chef.
4. **fastapi-tutorial**: Provision PostgreSQL and the application using a dedicated user, vaulted database credentials, pinned source/dependencies, systemd hardening, and an application health check. Coordinate with the application owner.
5. **Integration/cutover**: Run all roles together on a clean VM, compare ports/files/services to the Chef result, perform security and idempotence tests, then retire Chef/Berkshelf only after rollback procedures are documented.

### Assumptions

- The three directories containing `metadata.rb` and `recipes/default.rb` are the complete set of local cookbooks to migrate.
- The Chef run list in `solo.json` is the intended complete convergence scope.
- The Nginx templates and Fail2ban/sysctl files contain important behavior not visible in the reviewed recipe bodies and require detailed implementation review during the build phase.
- The `solo.json` Nginx document roots are authoritative over cookbook attribute defaults, but this must be confirmed with the service owner.
- `generic/fedora42` is an intended development target, although the current script strongly suggests an Ubuntu/Debian assumption; production OS is not specified.
- There is no cloud platform, external secret manager, certificate authority, monitoring system, or load balancer represented in this repository.
- The external FastAPI repository owns the application code and requirements; its release process and database migration strategy must be clarified.
- `redis_secure_password_123` and `fastapi_password` are examples or currently active credentials; they must be treated as compromised and rotated regardless.
- The intended external access is through Nginx on ports 80/443; direct FastAPI port 8000 access is not required unless confirmed.
- The three `.cluster.local` names require DNS or `/etc/hosts` handling; the commented Vagrant host entries and advertised addresses are inconsistent with the configured private IP and must be corrected.
- Self-signed certificates are acceptable only for local development, not for production.
- Migration estimates assume one environment and no major application refactoring; multi-environment promotion, HA Redis/PostgreSQL, certificate automation, or production compliance will increase scope.
- Team coordination should assign an Ansible/platform owner for roles and CI, an application owner for FastAPI/PostgreSQL, and a security reviewer for SSH, firewall, secrets, and certificates. Capture acceptance tests and rollback steps before changing the provisioning source of truth.
