# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant environment containing three local cookbooks: a multi-site Nginx web tier, caching services, and a FastAPI tutorial application with PostgreSQL. The target migration is moderate in operational risk despite the small module count because the repository embeds security hardening, TLS generation, firewall changes, application credentials, database initialization, and an application fetched from Git. A focused migration can be completed in approximately 2–4 weeks, including implementation, test coverage, security review, and parity testing; production hardening and certificate/secrets integration may extend that schedule.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web-server configuration for three static virtual hosts (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), each configured for HTTPS and HTTP-to-HTTPS redirection.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef
    - Key Features: Nginx service management, per-site document roots and index files, virtual-host templates, self-signed RSA certificates, TLS 1.2/1.3 settings, security headers, gzip, rate limits, Fail2ban, UFW, SSH hardening, and kernel network hardening.
    - Internal recipe structure: `default.rb` composes `nginx.rb`, `ssl.rb`, `sites.rb`, and `security.rb`; templates and static site files must be represented by Ansible templates/copy tasks.

- **cache**:
    - Description: Installs and enables Memcached and Redis, creates Redis logging storage, and configures a single Redis server on port 6379.
    - Path: `cookbooks/cache`
    - Technology: Chef
    - Key Features: External Memcached cookbook inclusion, RedisIO cookbook inclusion, Redis password authentication, Redis service enablement, and a post-generation configuration workaround that removes selected replication settings.

- **fastapi-tutorial**:
    - Description: Installs a Python FastAPI tutorial application from GitHub, creates a virtual environment, installs its requirements, provisions PostgreSQL, creates the application database/user, and runs the application with systemd and Uvicorn.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef
    - Key Features: Python 3 tooling, Git checkout of `https://github.com/dibanez/fastapi_tutorial.git` at `main`, PostgreSQL packages and service, database initialization, `.env` generation, and a root-owned systemd service listening on `0.0.0.0:8000`.

**CRITICAL PATH VERIFICATION:**
The module paths above were present in the supplied repository tree and their primary entrypoints were reviewed. No Puppet, Salt, or PowerShell modules were identified.

### Infrastructure Files

- `Berksfile`: Chef Supermarket and local cookbook dependency declarations. It references local `nginx-multisite`, `cache`, and `fastapi-tutorial`, plus `nginx ~> 12.0`, `memcached ~> 6.0`, and `redisio ~> 7.2.4`. Replace cookbook resolution with an Ansible collection/role dependency strategy and pin tested collection versions.
- `Vagrantfile`: Local Fedora 42 VM definition using the libvirt provider, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, and forwarded ports 80/443 to host ports 8080/8443. Retain or replace it with an Ansible-compatible test harness.
- `solo.json`: Chef run list and node data for all three cookbooks, site definitions, TLS locations, and security settings. Convert this to Ansible inventory/group variables and vaulted host/application variables.
- `solo.rb`: Chef Solo cache, cookbook search paths, and logging configuration. It has no direct Ansible equivalent beyond repository layout, logging, and execution configuration.
- `vagrant-provision.sh`: Installs Chef, Berkshelf, and cookbook dependencies, then runs Chef Solo. Replace with `ansible-playbook` provisioning or a Vagrant Ansible provisioner; remove the remote shell installation of Chef.
- `cookbooks/nginx-multisite/files/default/`: Static `index.html` content for the test, CI, and status sites. Move to role files or deploy from an application/content artifact.

### Target Details

- **Operating System**: The Vagrant box is `generic/fedora42`, but the provisioning script uses `apt-get`, and the cookbooks assume Debian/Ubuntu conventions such as `www-data`, UFW, `/var/log/auth.log`, and the `ssh` service name. The actual target OS is therefore unresolved. Establish an explicit supported target—preferably Fedora/RHEL-family tasks if Fedora 42 is authoritative, or Ubuntu if the existing package/service conventions are authoritative—and test all package names, paths, firewall modules, and service names.
- **Virtual Machine Technology**: libvirt via Vagrant, with rsync-based repository synchronization.
- **Cloud Platform**: Not specified. No AWS, Azure, or GCP-specific configuration was found.

## Migration Approach

### Key Dependencies to Address

- **Ansible core and collections**: Pin a supported `ansible-core` version and use `ansible.posix` for sysctl/firewall-adjacent tasks where appropriate, `community.general` for selected service/system tasks, and `community.postgresql` for database/user management. Validate collection availability and avoid shell implementations where idempotent modules exist.
- **Nginx** (Chef cookbook `~> 12.0`): Replace with an Ansible role using `ansible.builtin.package`, `template`, `copy`, `file`, `service`, and `community.crypto`/`openssl` handling. Preserve the four recipe responsibilities as role task files or clearly separated task blocks.
- **Memcached** (`~> 6.0`): Replace with distribution packages and an explicit service configuration role. Confirm whether Memcached is actually consumed by the application before exposing or enabling it.
- **RedisIO** (`~> 7.2.4`): Replace with distribution Redis packages or a vetted Redis role. Model the password through Ansible Vault and use a supported Redis configuration mechanism rather than the current file-edit workaround.
- **PostgreSQL**: Use distribution packages plus `community.postgresql.postgresql_user` and `postgresql_db`; ensure the Python PostgreSQL driver and local authentication rules are handled. Database creation must be idempotent and must not use `|| true`.
- **FastAPI/Python dependencies**: Keep the Git revision pinned to a commit or release rather than mutable `main`, create the virtual environment with Ansible, and install requirements with a controlled package policy. Consider artifact/container deployment for production.
- **systemd**: Replace the inline Chef service file with a version-controlled Ansible template and `ansible.builtin.systemd_service`, including daemon reload and handlers.
- **Vagrant/libvirt**: Keep Vagrant only as a development integration environment, or add Molecule/libvirt scenarios for role testing and OS matrix validation.

### Security Considerations

- **Hardcoded credentials**: `cache` contains the Redis password `redis_secure_password_123`; `fastapi-tutorial` contains the PostgreSQL password `fastapi_password` in SQL commands and the generated `.env`. Move both to Ansible Vault or an external secret manager, rotate them, restrict file permissions, and avoid rendering secrets in logs.
- **TLS certificates and private keys**: The Nginx cookbook creates self-signed certificates and private keys under `/etc/ssl/certs` and `/etc/ssl/private`. Treat these as development-only defaults. Production should consume managed certificates or securely supplied certificate/key material, with private keys mode 0640 or stricter and controlled group membership.
- **TLS configuration**: Preserve TLS 1.2/1.3 and review the explicit cipher list, HTTP/2 behavior, HSTS, and certificate renewal. Validate modern cryptographic policy against the selected OS rather than copying the current list blindly.
- **Firewall and exposure**: UFW defaults to deny and allows SSH, HTTP, and HTTPS. Reconcile this with the Fedora target, forwarded ports, and the FastAPI service on port 8000. Do not expose 8000 publicly unless required; restrict Redis and Memcached to trusted interfaces and networks.
- **SSH hardening**: Root login and password authentication are disabled through `sed` edits. Implement this with Ansible SSH configuration management, validate the effective configuration before restarting SSH, and ensure an administrative key-based access path exists to avoid lockout.
- **Fail2ban and logs**: The configuration references Debian-style `/var/log/auth.log` and Nginx logs. Adapt paths and filters to the target OS, test jails before enabling them, and ensure log rotation/retention.
- **Kernel hardening**: The sysctl template disables redirects, source routing, IPv6, and ICMP responses and enables SYN protections. Review each setting against application/network requirements, apply with `ansible.posix.sysctl`, and document the operational impact of disabling IPv6.
- **Application privilege**: The FastAPI systemd service runs as `root`. Create a dedicated non-privileged service account, set a writable application/runtime directory appropriately, and use systemd hardening options where compatible.
- **Supply chain**: The application is cloned from a mutable GitHub `main` branch and installs unconstrained requirements. Pin source and dependencies, verify provenance, and scan dependencies before production deployment.

### Technical Challenges

- **OS mismatch**: Fedora 42 is declared while the provisioning script and recipes use Debian/Ubuntu package and path assumptions. Resolve the target OS first; use OS-family variables or separate supported role defaults and integration tests.
- **Chef dependency behavior**: `nginx-multisite` uses the external `nginx` cookbook in Berkshelf but its `metadata.rb` does not declare that dependency, while the local `cache` cookbook declares `memcached` and `redisio`. Reconstruct the effective dependency graph and test the Ansible replacement independently rather than assuming Chef cookbook behavior.
- **Configuration path inconsistency**: `solo.json` defines site roots under `/var/www/...`, while cookbook attributes default to `/opt/server/...`; static files are copied to the node-derived roots. Decide which source of truth is intended and verify content placement before cutover.
- **Nginx template validity and ordering**: The site template contains conditional ERB that must be rendered into valid two-server-block configuration. Add `nginx -t` validation before reload and use handlers to coalesce changes.
- **Certificate lifecycle**: Self-signed generation is guarded only by file existence and has no renewal or trust distribution. Define development versus production certificate ownership and renewal responsibilities.
- **Database idempotency**: Shell commands use `|| true`, which hides failures and can leave partially configured users/databases. Replace with database modules, explicit privileges, and migration checks.
- **Application readiness**: The service starts after PostgreSQL but no health check or schema migration is defined. Add readiness validation, application smoke tests, and an explicit deployment/rollback strategy.
- **Security behavior changes**: Current rules include a 1 KB client body limit and aggressive ICMP/IPv6 restrictions. Confirm these are intentional and test application compatibility before preserving them.

### Migration Order

1. **Foundation and test harness**: Confirm the supported OS, inventory model, Vault/external secret mechanism, Ansible version, Vagrant/libvirt workflow, and baseline smoke tests.
2. **cache**: Migrate Memcached and Redis with network binding, authentication, service checks, and secret rotation. This is comparatively isolated but must be validated before application use.
3. **fastapi-tutorial**: Provision the service account, pinned application source, Python environment, PostgreSQL database, vaulted credentials, systemd unit, and health checks. Validate database connectivity and application startup.
4. **nginx-multisite**: Migrate Nginx, static content, virtual hosts, TLS, firewall, Fail2ban, SSH, sysctl, and security headers. This is the highest-impact module because it changes network access and security posture.
5. **Integration and cutover**: Run end-to-end tests for all three hostnames, HTTP-to-HTTPS redirects, certificate behavior, FastAPI availability, Redis/PostgreSQL connectivity, firewall rules, reboot persistence, and rollback.

Estimated implementation effort is roughly 2–3 engineer-weeks for a tested development migration, plus 1 week for security review, OS compatibility testing, documentation, and cutover preparation.

### Assumptions

- Chef Solo is the authoritative configuration mechanism; no Chef Server, roles, environments, encrypted data bags, or external secret store were found in the reviewed files.
- The three local cookbooks are the complete module inventory for this repository.
- `solo.json` is intended to override cookbook defaults, although its Nginx document roots conflict with `attributes/default.rb` and must be resolved.
- The Vagrant Fedora 42 box is the intended development target, but the apt-based provisioning script may be stale or copied from an Ubuntu workflow.
- The production operating system, host inventory, DNS, certificate authority, and cloud/platform deployment model are not specified.
- The three `.cluster.local` names require DNS or `/etc/hosts` entries; the commented Vagrant host entries are not active.
- The FastAPI application repository and its `requirements.txt` are external and were not audited here; its runtime, migrations, and health endpoint need confirmation.
- PostgreSQL is expected to run locally on the same host as FastAPI, based on the local connection string and service ordering.
- Redis and Memcached consumers, bind addresses, persistence requirements, and availability requirements are unspecified.
- Self-signed certificates are acceptable only for development; production certificate procurement and renewal are outside the current repository.
- The desired Ansible operating model—single playbook, reusable roles, or collection—is not specified; this plan assumes reusable roles with inventory/group variables and vaulted secrets.
- No backup, monitoring, alerting, log shipping, CI pipeline, or disaster-recovery configuration was found; these should be addressed before production migration.
- Team coordination should assign separate owners for OS/platform, Nginx/security, data services, and FastAPI/application migration; agree on variable naming, secret ownership, test acceptance criteria, and a rollback window before parallel implementation.
