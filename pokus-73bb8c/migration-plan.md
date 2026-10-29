# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant deployment containing three local cookbooks and four referenced external cookbooks. The target Ansible implementation is moderate in scope: the number of cookbooks is low, but the nginx multisite, firewall/SSH hardening, TLS behavior, Redis workaround, PostgreSQL bootstrap, and application service require careful idempotence and security redesign. A realistic delivery estimate is approximately 2–4 weeks for one engineer, or 1–2 weeks of implementation plus 1–2 weeks of validation and security review for a coordinated team. This assumes the current development behavior is the required target and that production certificate, DNS, database, and application requirements can be clarified during discovery.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx reverse proxy/web server configuration for three SSL-enabled virtual hosts (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), including document roots, static landing pages, security headers/configuration, TLS certificate locations, fail2ban, UFW, sysctl tuning, and SSH hardening.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Four-recipe orchestration (`security`, `nginx`, `ssl`, and `sites`), templated nginx configuration, enabled-site symlinks, development self-signed certificates, fail2ban, UFW default-deny rules, and disabling root/password SSH authentication.

- **cache**:
    - Description: Installs and enables Memcached and Redis, creates the Redis log directory, and configures a Redis server on port 6379 with password authentication.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: External `memcached` and `redisio` cookbook usage, Redis authentication, Redis enablement, and a post-configuration workaround that removes selected replication/buffer directives from `/etc/redis/6379.conf`.

- **fastapi-tutorial**:
    - Description: Installs a Python FastAPI tutorial application from GitHub, creates a virtual environment, installs its requirements, configures PostgreSQL, creates the application database/user, writes a `.env` file, and runs Uvicorn through systemd.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Python 3 tooling, Git checkout of `https://github.com/dibanez/fastapi_tutorial.git` at `main`, PostgreSQL service/database bootstrap, systemd unit, port 8000 binding, and automatic service enablement/start.

**CRITICAL PATH VERIFICATION:**
The three cookbook paths above are present in the supplied repository tree and their primary entrypoints and metadata were reviewed. No Puppet modules, Chef recipes outside these cookbooks, PowerShell modules, or Salt states were identified.

### Infrastructure Files

- `Berksfile`: Defines the local cookbooks and external Supermarket dependencies. In Ansible, replace Berkshelf resolution with a `requirements.yml` for collections/roles and explicit role/task ownership.
- `Vagrantfile`: Defines a `generic/fedora42` VM, libvirt provider, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, HTTP/HTTPS forwarded ports, and an rsynced repository. It is the principal local test harness, but its Fedora base conflicts with several Debian/Ubuntu assumptions in the Chef code.
- `solo.json`: Chef run list and node data for sites, SSL paths, fail2ban/UFW, and SSH settings. This should become group/host variables with secrets separated into Ansible Vault or an external secret manager.
- `solo.rb`: Chef Solo cache, cookbook paths, and logging configuration. It has no direct Ansible equivalent beyond inventory, role paths, and logging defaults.
- `vagrant-provision.sh`: Installs packages, Chef, Berkshelf, vendors dependencies, accepts the Chef license, and runs Chef Solo. Replace with an Ansible provisioner or a bootstrap script that installs Ansible on the control host; avoid piping an unpinned remote installer into a target without validation.
- `cookbooks/nginx-multisite/templates/`: Nginx, site, security, fail2ban, and sysctl templates. Port these to Ansible role templates after reviewing rendered output and preserving required distribution-specific paths.
- `cookbooks/nginx-multisite/files/`: Static `index.html` content for CI, status, and test sites. Move into an Ansible role `files/` tree or deploy the real application artifacts through CI.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource for line insertion/replacement and backups. Replace with Ansible `ansible.builtin.lineinfile` or, preferably, managed configuration templates; preserve backup behavior only where operationally required.

### Target Details

- **Operating System**: The Vagrant box is Fedora 42, but cookbook metadata claims Ubuntu >=18.04 and CentOS >=7. The implementation uses `apt-get`, `ufw`, `www-data`, Debian-style PostgreSQL/package names, and an `ssh` service, so the effective assumptions are primarily Debian/Ubuntu. The Ansible design must choose a supported target (preferably a pinned Ubuntu LTS or explicitly supported Fedora/RHEL variant) and use OS-family variables rather than mixing conventions.
- **Virtual Machine Technology**: Vagrant with the libvirt provider; 2 vCPUs and 2 GB memory are specified.
- **Cloud Platform**: Not specified. No AWS, Azure, GCP, or cloud-init integration was identified.
- **Network/application endpoints**: The intended test VM address is `192.168.121.10`; host ports 8080/8443 forward to guest 80/443. The provisioning script also prints `192.168.56.10`, which is inconsistent and must be corrected.

## Migration Approach

Create an Ansible project with an inventory, `group_vars`, and separate roles such as `nginx_multisite`, `cache`, `fastapi_tutorial`, and optionally a shared `security_baseline`. Keep site data, service settings, and package names variable-driven. Use check mode, Molecule or equivalent VM tests, and rendered configuration comparisons against a known-good Chef run before cutover. The migration should preserve service ordering (security prerequisites, packages, configuration, handlers, then application startup) while replacing shell-heavy Chef resources with native Ansible modules.

### Key Dependencies to Address

- **Chef Solo/Berkshelf**: Replace `solo.json`, `solo.rb`, and Berkshelf execution with Ansible inventory, role defaults/vars, `requirements.yml`, and pinned Ansible collection versions.
- **`nginx` cookbook `~> 12.0`**: Replace with an Ansible nginx role or first-party package/template/service tasks. Confirm nginx package version, site layout, modules, and security template behavior.
- **`memcached` cookbook `~> 6.0`**: Replace with distribution packages and Ansible service/configuration tasks; confirm whether Memcached is actually consumed by the application.
- **`redisio` cookbook `~> 7.2.4`**: Replace with a supported Redis package/role and explicit configuration. Reproduce only the required settings rather than copying the Ruby post-edit workaround.
- **PostgreSQL packages and client tools**: Use Ansible package/service modules and `community.postgresql` for databases, roles, passwords, and grants. Install the collection and required Python driver on the managed host.
- **Python virtual environment and dependencies**: Use `ansible.builtin.git`, `ansible.builtin.pip`, and explicit Python interpreter selection. Pin the Git revision and ideally application dependency versions rather than tracking `main`.
- **systemd**: Use a managed unit template, `systemd_service`, daemon reload handlers, and least-privilege service ownership.
- **OpenSSL/self-signed certificates**: Use `community.crypto` for development certificates, or integrate an approved CA/ACME workflow for real environments. Do not generate production certificates as a side effect of a generic deployment role.
- **fail2ban, UFW, SSH, and sysctl**: Use native package/service, `community.general.ufw`, `ansible.posix.sysctl`, and `lineinfile`/templates. Validate firewall changes through console access because an incorrect rule can lock out the host.

### Security Considerations

- **Hardcoded Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Remove it from source, rotate it, and store the replacement in Ansible Vault or an external secret manager. Configure Redis binding, ACL/authentication, and network exposure explicitly.
- **Hardcoded PostgreSQL credentials**: The FastAPI cookbook embeds username `fastapi`, password `fastapi_password`, and a connection URL in both shell commands and `/opt/fastapi-tutorial/.env`. Treat these credentials as compromised, rotate them, use Vault/secret-manager values, restrict `.env` permissions, and avoid exposing passwords in process arguments or logs.
- **TLS private keys and certificates**: Keys are written under `/etc/ssl/private` with `root:ssl-cert` and mode `0710` on the directory and `0640` on generated keys. Preserve strict ownership/permissions, preferably deploy CA-issued certificates through a controlled secret/certificate pipeline. The current self-signed certificates are development-only and cause browser warnings.
- **SSH hardening**: Root login and password authentication are disabled. Implement this with validated `sshd_config` templates/drop-ins, ensure authorized keys and console access exist first, validate syntax with `sshd -t`, and restart only through a controlled handler.
- **Firewall exposure**: UFW defaults to deny and allows SSH, HTTP, and HTTPS. Confirm whether PostgreSQL, Redis, Memcached, and FastAPI port 8000 must remain localhost-only; do not expose them merely because the application listens on `0.0.0.0`.
- **Application privilege**: The generated systemd service runs FastAPI as `root`. Create a dedicated non-root application account, restrict the working tree and `.env`, and bind through nginx or a private interface where possible.
- **Supply-chain risk**: The Git checkout tracks the mutable `main` branch, and the provisioning script downloads and executes Chef's installer via `curl`. Pin source revisions, validate package repositories/signatures, and establish an approved artifact process during migration.
- **Configuration integrity**: Replace shell `sed`, `execute`, and ad hoc file edits with templated, validated files. Use Ansible Vault for all credential material and document ownership/access boundaries for the operations team.

### Technical Challenges

- **OS mismatch**: Fedora 42 is the Vagrant target while the recipes use `apt-get`, UFW, Debian package names, `www-data`, and likely Debian service paths. Decide whether to standardize on Ubuntu/Debian or port fully to Fedora/RHEL, then test package names, firewall tooling, service names, nginx paths, and PostgreSQL behavior.
- **Nginx template fidelity**: The migration must preserve multiple virtual hosts, static content, SSL paths, security headers, reload handlers, and removal of the default site. Render and validate with `nginx -t` before reload.
- **Certificate lifecycle**: The current recipe creates certificates only when files are absent and has no renewal or trust model. Define development versus production certificate ownership, renewal, SAN requirements, and rollback behavior.
- **Redis configuration workaround**: The Ruby block removes several generated directives after `redisio` runs. Determine the underlying Redis/version compatibility issue and express the supported configuration directly; do not blindly reproduce brittle text manipulation.
- **Database idempotence**: PostgreSQL creation currently uses SQL commands followed by `|| true`, which hides failures and does not safely manage password changes. Use database/role modules, explicit privilege state, and migration checks.
- **Application deployment repeatability**: Installing from mutable Git `main`, running pip on every converge, and starting as root are operational risks. Define release/version strategy, dependency locking, health checks, migrations, and rollback expectations.
- **Firewall and SSH cutover risk**: Security tasks can sever access or interrupt nginx traffic. Apply in staged runs with console fallback, validation, and handlers; test in the Vagrant VM before broader rollout.
- **Data/configuration ambiguity**: `solo.json` document roots (`/var/www/...`) differ from cookbook attribute defaults (`/opt/server/...`), and the static files are selected by hostname prefix. Confirm the intended source of truth before converting variables.

### Migration Order

1. **Foundation and test harness**: Pin the target OS, repair the Vagrant network/address inconsistency, create inventory and CI validation, and establish Ansible collections and secret handling.
2. **cache**: Migrate Memcached/Redis package, service, log directory, and secure credentials. This is relatively isolated but should be validated before application integration.
3. **fastapi-tutorial**: Migrate Python runtime, pinned application checkout, PostgreSQL role/database, protected environment configuration, non-root systemd unit, and health checks.
4. **nginx-multisite**: Migrate nginx/templates, static site files, virtual hosts, certificates, and reload behavior; integrate upstream routing if the current templates proxy to FastAPI.
5. **security baseline and hardening**: Apply UFW, fail2ban, sysctl, SSH, file permissions, and certificate controls after functional services work, using staged rollout and lockout validation.
6. **Cutover and decommissioning**: Compare rendered configuration and service behavior with Chef, run idempotence checks twice, document rollback, then retire Chef/Berkshelf provisioning and update team runbooks.

### Assumptions

- The repository is a development/demo deployment rather than a production-ready platform; no production inventory, CI pipeline, monitoring, backup policy, DNS automation, or certificate authority integration was supplied.
- The three directories under `cookbooks/` are the complete local cookbook inventory; external cookbooks are resolved by Berkshelf and their internals were not present for review.
- `solo.json` is intended to override cookbook defaults, but the differing document-root paths are unresolved and must be confirmed.
- The intended operating system is unresolved: Vagrant specifies Fedora 42, while recipes and metadata primarily indicate Ubuntu/Debian compatibility with some CentOS claims.
- The FastAPI repository, its `requirements.txt`, schema migrations, runtime behavior, and health endpoint were not included; application-specific deployment requirements require owner confirmation.
- The current FastAPI service intentionally listens on all interfaces and runs as root; the plan assumes security hardening changes are acceptable.
- The Redis and PostgreSQL passwords shown in source are examples or existing credentials; they must nevertheless be treated as exposed and rotated.
- The self-signed certificates are for development. Production certificate issuance, renewal, SANs, and trust distribution are not specified.
- DNS/hosts resolution for the three cluster names is not implemented in the active Vagrant configuration; test clients must supply suitable name resolution.
- The provisioning output references `192.168.56.10`, while the Vagrant private address is `192.168.121.10`; this is assumed to be a documentation bug.
- The commented Chef provisioner and active shell provisioner indicate Chef Solo is the current execution model; no Chef Server, environments, roles, encrypted data bags, or Chef Vault usage were identified.
- Team coordination should assign separate owners for OS/platform compatibility, nginx/security, cache/database, and FastAPI application behavior; agree on variable naming, secret ownership, review gates, and a single tested inventory before merging roles.
