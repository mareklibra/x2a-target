# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant environment containing three local cookbooks: an Nginx multi-site/security stack, caching services, and a FastAPI tutorial application with PostgreSQL. The target migration is moderate in operational risk despite the small module count because it combines web serving, firewall and SSH hardening, TLS key material, databases, Redis authentication, and an application pulled from GitHub. A focused implementation can likely be completed in **1–2 weeks** by one engineer, including validation and documentation; allow **2–3 weeks** if production-grade secret management, certificate issuance, OS portability, and CI testing are required.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Installs and configures Nginx for three virtual hosts (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), creates their document roots and static index pages, enables HTTPS, and applies host security hardening.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef
    - Key Features: Nginx service management, per-site configuration and symlinks, self-signed RSA certificates, Fail2ban, UFW default-deny firewall, sysctl settings, disabled root SSH login, and disabled SSH password authentication.

- **cache**:
    - Description: Configures Memcached and a single Redis server through external Chef cookbooks, including a Redis password and a local post-configuration cleanup workaround.
    - Path: `cookbooks/cache`
    - Technology: Chef
    - Key Features: Memcached inclusion, Redis on port 6379, authenticated Redis configuration, Redis log directory, Redis enablement, and direct editing of the generated Redis configuration.

- **fastapi-tutorial**:
    - Description: Installs the Python/PostgreSQL prerequisites for a FastAPI tutorial, checks out the application from GitHub, creates a Python virtual environment, installs requirements, provisions a PostgreSQL database/user, writes an application `.env`, and runs the application under systemd with Uvicorn.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef
    - Key Features: Git-based deployment from `https://github.com/dibanez/fastapi_tutorial.git` at `main`, Python virtualenv, PostgreSQL service/database setup, systemd unit, Uvicorn on `0.0.0.0:8000`, and automatic service restart.

**CRITICAL PATH VERIFICATION:**
The module paths above were taken directly from the provided repository tree and the corresponding `metadata.rb` and primary `recipes/default.rb` files. No external cookbook is listed as a local module.

### Infrastructure Files

- `Berksfile`: Defines the Chef Supermarket source, the three local cookbooks, and external `nginx`, `memcached`, and `redisio` cookbook dependencies. Replace this dependency graph with Ansible collections/roles and system packages.
- `Vagrantfile`: Defines a `generic/fedora42` VM, libvirt provider, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, and HTTP/HTTPS forwarded ports. It rsyncs the repository to `/chef-repo` and invokes the shell provisioner. Rework it for the Ansible provisioner or use it only as a test harness.
- `solo.json`: Chef run list and node attributes for all three cookbooks, including site names, document roots, TLS directories, and security settings. Its hardcoded credentials and environment values must not be carried into Ansible plaintext vars.
- `solo.rb`: Chef Solo cache, cookbook search paths, and logging configuration. It has no direct Ansible equivalent beyond inventory, project layout, and logging settings.
- `vagrant-provision.sh`: Installs Chef and Berkshelf, downloads/vendors dependencies, and runs Chef Solo. Replace with Ansible installation/invocation or remove it after the Vagrant workflow is converted.
- `cookbooks/nginx-multisite/templates/`: Nginx, security, Fail2ban, sysctl, and virtual-host templates that will become Jinja2 templates.
- `cookbooks/nginx-multisite/files/`: Static HTML fixtures for the three sites; preserve as Ansible `files/` content or deploy application-owned content separately.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource. It is not used by the reviewed recipes, but any future use should map to `ansible.builtin.lineinfile` with explicit validation and backup behavior.

### Target Details

- **Operating System**: The Vagrant box is explicitly `generic/fedora42`, but the provisioning script uses `apt-get`, and the cookbooks support Ubuntu >= 18.04 and CentOS >= 7. The effective tested platform is therefore ambiguous and currently inconsistent. Establish a supported target before implementation; for this plan, default to **Red Hat Enterprise Linux 9 / Fedora-compatible systems** unless the application environment is intentionally standardized on Ubuntu. On RHEL-family hosts, replace UFW with firewalld or explicitly install/use UFW only if that is an approved requirement.
- **Virtual Machine Technology**: libvirt via Vagrant, with rsync-based source synchronization.
- **Cloud Platform**: Not specified. No AWS, Azure, or GCP integration is present.

## Migration Approach

Create an Ansible project with inventories for development and target environments, roles for `nginx_multisite`, `cache`, and `fastapi_tutorial`, and a top-level site playbook. Keep site definitions, ports, package names, and service names in group/host variables. Convert Chef templates to Jinja2 only after comparing their full contents and testing `nginx -t`, `sshd -t`, and sysctl behavior.

### Key Dependencies to Address

- **Chef Supermarket `nginx` `~> 12.0`**: Replace with `ansible.builtin.package`, `service`, `template`, `file`, and `community.general`/platform-specific firewall tasks. Use the distribution's Nginx package and pin it through OS repositories or an approved repository policy.
- **Chef Supermarket `memcached` `~> 6.0`**: Replace with package installation, service enablement, and an explicit Memcached configuration role. Confirm bind address and exposure requirements rather than reproducing defaults blindly.
- **Chef Supermarket `redisio` `~> 7.2.4` and `redisio::enable`**: Replace with an Ansible Redis role or carefully managed distribution package/service tasks. Preserve authentication and validate the Redis configuration; avoid an unreviewed direct-file cleanup workaround.
- **Python 3, pip, venv, Git, PostgreSQL, PostgreSQL contrib, and libpq development headers**: Map to distribution packages and `ansible.builtin.git`, `ansible.builtin.pip`, `community.postgresql.postgresql_db`, and `community.postgresql.postgresql_user`.
- **FastAPI repository and `requirements.txt`**: Pin a reviewed Git commit or release rather than tracking `main`; install requirements in the virtual environment and add deployment checks.
- **systemd**: Use a Jinja2-managed unit or `ansible.builtin.systemd_service`, with daemon reload and handlers.
- **OpenSSL/self-signed certificates**: Use `community.crypto` to generate development certificates, or integrate the approved CA/ACME process for non-development environments.
- **Chef custom resource and shell commands**: Replace with idempotent Ansible modules. Avoid broad shell commands unless no module exists and add `changed_when`/`creates`/validation conditions.

### Security Considerations

- **Hardcoded Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Move it to Ansible Vault or an external secret manager, rotate it during migration, and restrict Redis network binding and ACLs.
- **Hardcoded PostgreSQL credentials**: The FastAPI cookbook creates user `fastapi` with password `fastapi_password` and writes the same credential into `/opt/fastapi-tutorial/.env` with mode `0644`. Treat it as compromised, rotate it, use Vault-backed variables, set restrictive ownership/mode, and use PostgreSQL least privilege.
- **Database command safety**: The Chef recipe embeds SQL and uses `|| true`, which can conceal failed creation and is not safely idempotent. Use PostgreSQL Ansible modules and explicit state checks.
- **TLS certificates and private keys**: Self-signed certificates are generated for all enabled sites for 365 days under `/etc/ssl/certs` and `/etc/ssl/private`; private keys are assigned `root:ssl-cert` and mode `0640`. Do not regenerate production keys casually. Store supplied keys securely, use Vault or a certificate manager, and define renewal/rotation ownership.
- **SSH hardening**: Root login and password authentication are disabled. Validate that inventory users have working SSH keys and sudo before applying the change, and use `ansible.builtin.sshd`-appropriate templating/validation with a safe restart handler.
- **Firewall and service exposure**: The source opens SSH, HTTP, and HTTPS but the FastAPI service listens on all interfaces at port 8000 and no explicit firewall rule is shown for it. Decide whether Uvicorn should be localhost-only behind Nginx or whether 8000 is intentionally exposed.
- **Fail2ban and sysctl**: Preserve the intended jail and kernel hardening only after reviewing the source templates for distribution compatibility. Test against the target OS and avoid applying settings that disrupt containerized or virtualized workloads.
- **Supply-chain and remote code**: The application is cloned from an unpinned `main` branch and pip dependencies are installed remotely. Pin source revisions and review/hash-lock dependencies where practical.
- **Certificate warning and test data**: The provisioning script explicitly expects self-signed certificate warnings. Keep this behavior confined to development inventory.

### Technical Challenges

- **OS mismatch**: Fedora 42 is provisioned with Debian/Ubuntu `apt-get`, while cookbook metadata supports Ubuntu and CentOS. Resolve the platform matrix first, then use OS-family variables and conditional package/service/firewall tasks.
- **Conflicting configuration sources**: `solo.json` sets site roots under `/var/www/...`, while `attributes/default.rb` defaults to `/opt/server/...`. The current run likely uses the JSON values, but this must be made explicit in Ansible variables and tested with the static file source layout.
- **Static file path assumptions**: Site content is selected from the first label of each hostname (`test`, `ci`, `status`). Preserve this mapping explicitly rather than relying on string parsing.
- **TLS behavior**: Certificates are generated only if both files do not exist and are tied to hostnames. Define certificate ownership, SANs, renewal, and production-vs-development behavior.
- **Redis workaround**: The Ruby block removes replication-related directives after `redisio` generates its configuration. Determine why those directives are present and model the desired Redis configuration directly; do not blindly port the text replacement.
- **Application lifecycle**: The cookbook installs as root, runs Uvicorn as root, follows `main`, and has no migration/health-check strategy. Introduce a dedicated service account, pinned artifact, health checks, controlled restart behavior, and database migration policy.
- **Idempotence and failure masking**: Several `execute` resources use shell commands and the database commands suppress errors. Ansible tasks should report meaningful changes/failures and validate configuration before service restarts.
- **Networking and test naming**: The Vagrant host entries are commented out, while hostnames are used for virtual-host and certificate configuration. Add deterministic test DNS/hosts entries and test both forwarded and private-network access.
- **Template fidelity**: The full ERB templates were not required for this survey. Their directives, variables, and distribution assumptions need a focused implementation review before conversion.

### Migration Order

1. **Foundation and test harness**: Resolve the OS target, convert Vagrant/provisioning to invoke Ansible, create inventories, establish Vault/secret storage, and add linting and molecule or equivalent VM tests.
2. **cache**: Implement Memcached and Redis package/service configuration with secret rotation and network restrictions. Validate independently because the current Redis workaround is specialized.
3. **fastapi-tutorial**: Provision PostgreSQL, application files, virtualenv, systemd, and a non-root service account. Add application health checks and verify database connectivity.
4. **nginx-multisite static content**: Deploy Nginx, document roots, static fixtures, site templates, and service handlers.
5. **nginx-multisite security and TLS**: Apply firewall, SSH, sysctl, Fail2ban, and certificate management only after connectivity and recovery tests. Run Nginx/SSH configuration validation before reloads.
6. **Integration and cutover**: Test all three hostnames, HTTPS certificates, reverse-proxy/application behavior, Redis/PostgreSQL connectivity, reboot persistence, and failure recovery; then retire Chef/Berkshelf provisioning.

### Assumptions

- The three local directories under `cookbooks/` are the complete module inventory.
- No Chef roles, environments, encrypted data bags, Chef Vault items, or external application repositories are present beyond the files shown.
- The external cookbooks are dependencies only; their complete behavior and templates were not inspected and must be verified during implementation.
- `solo.json` is intended to override the cookbook defaults, particularly the `/var/www` document roots.
- The three hostnames are development/test virtual hosts, and self-signed TLS is not acceptable for production.
- The Redis and PostgreSQL passwords in source are placeholders or development credentials, but they must still be treated as exposed and rotated.
- The desired target OS is not settled. Package names, firewall technology, SSH service naming, and PostgreSQL/Redis paths require a platform decision.
- The FastAPI application is expected to listen behind Nginx, although the source binds it to all interfaces and does not show an Nginx upstream configuration.
- Port 8000 exposure, database schema migration, backup/restore, monitoring, logging retention, and operational alerting are not defined in the reviewed files and need owner decisions.
- The exact contents of the ERB templates and static files remain implementation inputs; behavior must be regression-tested rather than inferred solely from filenames.
- A team should assign separate owners for platform/Ansible structure, security/secrets, database and cache services, application deployment, and integration testing, with a shared acceptance checklist and rollback plan.
