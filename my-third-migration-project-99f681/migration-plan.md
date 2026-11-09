# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant deployment composed of three local cookbooks and several external cookbook dependencies. The target state provisions an Nginx multi-site front end, host security controls, Redis and Memcached, and a FastAPI tutorial application backed by PostgreSQL. The migration is moderate in breadth but has several security and portability concerns: credentials are embedded in cookbook code, the application runs as root, TLS certificates are development self-signed certificates, and the Vagrant base image is Fedora while the provisioning script assumes Debian/Ubuntu tooling. A focused implementation can likely be completed in approximately 1–2 weeks, including validation and security remediation; production hardening and application deployment design may extend that to 2–4 weeks.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx reverse-proxy/web hosting configuration for three named sites (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), including document roots, virtual hosts, SSL paths, and static index content.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Nginx installation and service management, generated per-site configuration, enabled-site symlinks, security headers/configuration templates, static site files, and self-signed certificate generation.
    - Migration scope: Convert the four included recipes (`nginx`, `security`, `ssl`, and `sites`), attributes, ERB templates, static files, and the custom `lineinfile` resource into Ansible roles/tasks, templates, handlers, files, and variables.

- **cache**:
    - Description: Host caching services configuration for Memcached and a single Redis instance listening on port 6379, with Redis authentication and service enablement.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: Inclusion of external Memcached and Redis cookbook recipes, Redis log directory creation, Redis password configuration, and a post-configuration workaround that removes selected replication-related directives.
    - Migration scope: Replace external cookbook behavior with Ansible package/service/configuration tasks or supported Ansible collections; model the Redis configuration declaratively and remove or explicitly justify the current post-edit workaround.

- **fastapi-tutorial**:
    - Description: Installs and launches the FastAPI tutorial application from GitHub, creates a Python virtual environment, installs requirements, configures local PostgreSQL, writes an environment file, and runs Uvicorn as a systemd service.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Python 3 tooling, Git checkout of `https://github.com/dibanez/fastapi_tutorial.git` at `main`, virtualenv creation, PostgreSQL packages/service, database and user creation, `.env` generation, and systemd service management on port 8000.
    - Migration scope: Use Ansible package, git, `ansible.builtin.pip`, PostgreSQL modules, `template`/`copy`, and `systemd` tasks. Add idempotent database/user management and a non-root application service as part of the migration rather than reproducing shell-command behavior verbatim.

**CRITICAL PATH VERIFICATION:**
All three module paths above were present in the provided repository tree and their primary entrypoints were reviewed. No Puppet modules, Salt states, or PowerShell modules were identified.

### Infrastructure Files

- `Berksfile`: Chef dependency source and lock-free version constraints. It references the three local cookbooks and external `nginx ~> 12.0`, `memcached ~> 6.0`, and `redisio ~> 7.2.4` cookbooks. The commented `ssl_certificate ~> 2.1` dependency is not active. Replace these dependencies with Ansible roles/collections and pinned package/application versions.
- `Vagrantfile`: Local test orchestration using `generic/fedora42`, hostname `chef-nginx`, libvirt, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, and ports 80/443 forwarded to host ports 8080/8443. Reuse or update this as the Ansible acceptance harness.
- `vagrant-provision.sh`: Installs Chef and Berkshelf, vendors cookbook dependencies, and runs `chef-solo`. Replace it with Ansible installation/inventory/playbook execution or a Vagrant Ansible provisioner. It currently uses `apt-get`, which is inconsistent with the Fedora base image.
- `solo.json`: Chef run list and node attributes. It defines the three sites, SSL directories, Fail2ban/UFW enablement, and SSH hardening. Convert these values into group/host vars and role defaults, with secrets removed from plaintext.
- `solo.rb`: Chef Solo cache, cookbook paths, and logging configuration. It has no direct Ansible equivalent beyond playbook configuration, inventory, and logging conventions.
- `cookbooks/nginx-multisite/templates/`: Nginx, site, security, Fail2ban, and sysctl templates. Port these to Jinja2 only after reviewing their rendered behavior and validating syntax.
- `cookbooks/nginx-multisite/files/`: Static `index.html` content for the three sites. Move to role `files/` or deploy from an application artifact repository.

### Target Details

- **Operating System**: The cookbook metadata claims Ubuntu >=18.04 and CentOS >=7 support. The provisioning script explicitly uses Debian/Ubuntu `apt-get`, `ufw`, `www-data`, and Debian-style service/package expectations, while `Vagrantfile` selects `generic/fedora42`. The effective target is therefore ambiguous. Establish a supported target first; default migration testing should use Ubuntu 22.04/24.04 unless Fedora compatibility is a firm requirement. If no decision is available, use Red Hat Enterprise Linux 9 as the enterprise default and replace UFW with firewalld, while preserving a separate Ubuntu variable set.
- **Virtual Machine Technology**: libvirt via Vagrant, with rsync synchronization into `/chef-repo`. No production virtualization platform is specified.
- **Cloud Platform**: Not specified. No AWS, Azure, or GCP configuration was identified.

## Migration Approach

### Key Dependencies to Address

- **Chef Solo/Berkshelf**: Replace `chef-solo`, Berkshelf vendoring, `solo.rb`, and `solo.json` with an Ansible project containing inventory, group/host variables, roles, and a documented `ansible-playbook` command. Pin `ansible-core` and collection versions in a requirements file.
- **nginx cookbook `~> 12.0`**: Replace with explicit Ansible package, template, site enablement, and service-handler tasks, or a vetted Ansible Nginx role. Preserve the repository's custom templates and validate with `nginx -t`.
- **memcached cookbook `~> 6.0`**: Use OS packages and Ansible service/configuration tasks, or a vetted collection/role. Define listen address, memory, and access controls explicitly rather than relying on cookbook defaults.
- **redisio cookbook `~> 7.2.4`**: Replace with an Ansible Redis role or native tasks. Reconcile the current `ruby_block` edits with the Redis version's supported configuration format and model the desired settings declaratively.
- **Python/Git dependencies**: The FastAPI repository's `requirements.txt` is external and tracks the mutable `main` branch. Pin a commit/tag and preferably validate dependency hashes or use a controlled artifact source.
- **PostgreSQL**: Use `community.postgresql` with a local or explicitly configured connection, idempotent database/user modules, and a vaulted password. Confirm the required PostgreSQL major version and authentication policy.
- **systemd**: Use an Ansible `template` for the FastAPI unit, `daemon_reload`, and `systemd`/`service` handlers. Define a dedicated service account and least-privilege filesystem ownership.
- **TLS certificates**: The current implementation uses OpenSSL-generated self-signed certificates. For production, use centrally managed certificates or ACME automation; retain self-signed generation only for isolated development.

### Security Considerations

- **Plaintext Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Move it to Ansible Vault or an external secrets manager, rotate it, and ensure Redis is not exposed beyond trusted interfaces.
- **Plaintext PostgreSQL credentials**: The FastAPI cookbook embeds `fastapi_password` in SQL commands and writes it into `/opt/fastapi-tutorial/.env` with mode `0644`. Rotate the password, use Vault, set restrictive permissions (for example 0600), and avoid exposing secrets in command-line process listings or logs.
- **Database creation behavior**: Shell commands use `|| true`, masking failures and making state difficult to verify. Replace them with idempotent PostgreSQL modules and explicit privilege management.
- **TLS private keys**: Private keys are placed under `/etc/ssl/private` with `root:ssl-cert` ownership and mode 0710 on the directory, but certificates are self-signed and generated with a fixed subject pattern. Protect keys with Ansible Vault or a certificate manager where applicable, and verify permissions and renewal strategy.
- **SSH hardening**: The source disables root login and password authentication through direct `sed` edits. Use `ansible.builtin.sshd`-appropriate templating or a controlled drop-in, validate with `sshd -t` before restarting, and ensure authorized key access and console recovery are available.
- **Firewall and Fail2ban**: UFW is enabled with default deny and ports 22/80/443 allowed. Preserve the intended policy, but make it OS-specific; on Fedora/RHEL use firewalld/nftables. Review Fail2ban jail settings and ensure the firewall is not enabled before the required management access path is confirmed.
- **Nginx exposure**: Validate all virtual host headers, TLS protocols/ciphers, security headers, access logs, and upstream behavior. The application itself listens on `0.0.0.0:8000`; decide whether it should be bound to localhost and proxied by Nginx instead of directly exposed.
- **Root application service**: The systemd unit runs Uvicorn as `root`. Create a dedicated application user, constrain permissions, and consider systemd hardening (`NoNewPrivileges`, filesystem protections, and resource limits).
- **Supply-chain risk**: Chef installation is fetched by piping a remote script to `bash`; the Git repository tracks `main`; and Python requirements are installed at provisioning time. Pin and verify all external inputs, preferably through CI and an internal mirror.
- **Custom configuration resource**: The custom `lineinfile` resource builds regular expressions from input and writes timestamped backups. It is not used by the reviewed primary recipes, but should not be carried forward unless a concrete use is found; use Ansible modules instead.

### Technical Challenges

- **OS mismatch and portability**: Fedora 42 is the Vagrant image, but `apt-get`, UFW, `www-data`, and metadata support statements suggest Debian/Ubuntu. Resolve the platform matrix before implementation and add molecule/Vagrant tests for each supported family.
- **Template semantic parity**: Nginx, Fail2ban, and sysctl templates are behaviorally important but were not represented as native Ansible defaults. Port them carefully, test rendered configuration, and use handlers rather than unconditional reloads.
- **Certificate lifecycle**: Current certificates are generated once when absent and have one-year validity. Define whether development certificates, pre-provisioned secrets, or ACME certificates are required and test renewal/reload behavior.
- **Redis workaround**: The Chef `ruby_block` removes several replication directives after the external cookbook writes configuration. This may reflect an old Redis/cookbook incompatibility. Identify the intended Redis topology and replace the workaround with a supported configuration.
- **Application reproducibility**: A mutable Git branch and unconstrained application dependencies can make Ansible runs produce different systems. Pin source and dependency versions before acceptance testing.
- **Idempotence and ordering**: The application requires PostgreSQL, the virtual environment, its `.env`, and systemd in a defined order. Use task dependencies, handlers, and readiness checks; avoid shell commands that hide errors.
- **DNS and host-name testing**: The Vagrant host entries are commented out, while the Nginx sites use cluster-local names. Add deterministic test DNS or `/etc/hosts` setup and validate both forwarded ports and the private address.
- **Unclear application deployment contract**: The source installs and starts the API but does not define migrations, health checks, logs, or reverse proxy integration. Confirm operational requirements with the application owner.

### Migration Order

1. **Foundation and test harness**: Decide supported OS(s), create the Ansible inventory/variables layout, replace Chef provisioning in Vagrant, pin Ansible collections, and establish lint, syntax, idempotence, and service-health checks.
2. **nginx-multisite**: Migrate the lowest-level web and security configuration first, including static files, virtual hosts, Nginx validation, firewall policy, SSH hardening, Fail2ban, sysctl, and TLS strategy. This establishes the external access path for later application testing.
3. **cache**: Migrate Memcached and Redis with explicit bind/authentication settings, package/service checks, and secret handling. Validate that the Redis workaround is no longer necessary.
4. **fastapi-tutorial and PostgreSQL**: Migrate repository checkout, Python environment, database provisioning, secret file, and systemd service. Make the service non-root and pin the application revision before production use.
5. **Integration hardening and cutover**: Run repeated convergences, verify all three hostnames and TLS, test reboot persistence, review firewall lockout scenarios, compare rendered configurations, document rollback, and retire Chef/Berkshelf only after acceptance.

### Assumptions

- The repository is a development/demo environment rather than a production-ready service because it uses Vagrant, self-signed certificates, example domains, and embedded passwords.
- The three local cookbooks are the complete module inventory; no additional cookbook directories or policy files were provided.
- The external cookbook versions in `Berksfile` are intended dependencies, but their exact resolved versions are unknown because no Berkshelf lock file was provided.
- The desired application source is the FastAPI tutorial repository and not a private fork; this must be confirmed before pinning a commit.
- PostgreSQL is intended to run locally on the same host as the FastAPI service, with database `fastapi_db` and user `fastapi`.
- Redis is intended as a single local server, not a replication or clustered deployment; the replication directive removal suggests unresolved historical configuration behavior.
- The three Nginx sites should continue serving their static cookbook-provided index pages, although the attribute defaults and `solo.json` document roots differ. The JSON values should take precedence only after confirmation.
- SSL paths are intended to remain `/etc/ssl/certs` and `/etc/ssl/private`, but certificate ownership, trust requirements, and renewal ownership are not fully defined.
- UFW is the intended firewall on Ubuntu; Fedora/RHEL support requires a deliberate firewalld/nftables implementation rather than a literal port.
- The Vagrant private IP is `192.168.121.10`, while the provisioning script prints `192.168.56.10`; the correct address and expected access URLs must be corrected.
- No cloud provider, inventory of production hosts, CI pipeline, monitoring, backup policy, database migration process, or DNS automation is specified.
- No Chef encrypted data bags or Chef Vault usage were identified in the reviewed files; the visible secrets are plaintext and must be treated as compromised and rotated.
- A practical delivery estimate is 1–2 weeks for a tested development migration and 2–4 weeks if production-grade certificate management, secrets integration, OS portability, application hardening, and operational runbooks are required.
- Migration coordination should assign separate owners for platform/Ansible structure, Nginx/security, caching, and FastAPI/PostgreSQL; require application and security review before cutover; and maintain a shared acceptance checklist covering idempotence, reboot behavior, access recovery, TLS, secrets, and rollback.
