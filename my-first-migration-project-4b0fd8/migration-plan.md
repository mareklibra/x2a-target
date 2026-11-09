# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant deployment with three local cookbooks and three external cookbook dependencies. The target migration is moderate in operational risk despite the small module count: the Nginx cookbook combines web serving, TLS, host security, firewall, fail2ban, and templating; the application cookbook embeds database credentials and an unpinned Git dependency; and the cache cookbook contains a plaintext Redis password plus a post-configuration workaround. A focused migration can likely be completed in approximately 2–4 engineering weeks, including validation on a clean VM, security remediation, and handover documentation. The estimate assumes the current development-only topology is the intended initial scope and that production requirements are confirmed before implementation.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx multi-site web serving for three virtual hosts, with static content, HTTP-to-HTTPS redirects, self-signed development certificates, hardened Nginx settings, host firewall rules, fail2ban, sysctl settings, and SSH hardening.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef
    - Key Features: `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`; per-site document roots and templates; TLS 1.2/1.3; security headers; rate limiting; UFW; fail2ban; disabled root SSH login and password authentication.
    - Ansible shape: A role or set of roles for Nginx, TLS, host security, and fail2ban, driven by a list of virtual hosts. Preserve ordering and handlers so Nginx reloads only after valid configuration changes.

- **cache**:
    - Description: Installs and enables Memcached and Redis, configures a Redis server on port 6379, creates its log directory, and applies a Ruby-based cleanup workaround to generated Redis configuration.
    - Path: `cookbooks/cache`
    - Technology: Chef
    - Key Features: External `memcached` and `redisio` cookbook use; Redis authentication; Redis enablement; direct manipulation of `/etc/redis/6379.conf` to remove replication-related directives.
    - Ansible shape: A cache role using distribution packages or explicitly selected Redis/Memcached roles, with Redis settings rendered declaratively and the workaround replaced by a supported configuration model or a narrowly scoped task.

- **fastapi-tutorial**:
    - Description: Installs Python and PostgreSQL prerequisites, clones a FastAPI tutorial application, creates a virtual environment, installs application requirements, provisions a PostgreSQL database and user, writes an environment file, and runs Uvicorn through systemd.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef
    - Key Features: Git checkout from `https://github.com/dibanez/fastapi_tutorial.git` at `main`; Python virtualenv; PostgreSQL service; `fastapi` database/user; systemd service on port 8000; application `.env` file.
    - Ansible shape: Separate application and PostgreSQL roles where possible, with `ansible.builtin.git`, package/virtualenv tasks, PostgreSQL modules, a systemd unit template, and Vault-managed database credentials. Pin the application revision and dependency inputs for repeatability.

### Infrastructure Files

- `Berksfile`: Defines local cookbook sources and external Chef Supermarket dependencies: `nginx ~> 12.0`, `memcached ~> 6.0`, and `redisio ~> 7.2.4`; the SSL cookbook is commented out. Replace this dependency resolution with Ansible roles/collections and explicitly managed package versions.
- `solo.rb`: Chef Solo configuration, including cookbook paths, cache location, and logging. Replace with an Ansible inventory, `ansible.cfg`, role paths, and execution conventions.
- `solo.json`: Chef run list and node attributes. It defines the three Nginx sites, certificate/key directories, and security switches. Convert these into group/host variables and role defaults, with secrets removed from plaintext variables.
- `Vagrantfile`: Defines a `generic/fedora42` VM, libvirt provider, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, and forwarded ports 8080/8443 to guest ports 80/443. The synced repository is mounted at `/chef-repo`; replace Chef provisioning with an Ansible provisioner or a documented host-to-VM workflow.
- `vagrant-provision.sh`: Uses `apt-get`, installs Chef and Berkshelf from Omnitruck/gems, vendors cookbook dependencies, and runs Chef Solo. Replace with Ansible installation/provisioning or use it only as a temporary bootstrap script. The script also documents self-signed certificate expectations.
- `cookbooks/nginx-multisite/templates/`: Nginx, security, virtual-host, sysctl, and fail2ban templates. Port these to Jinja2 and validate with `nginx -t`, `sysctl`, and fail2ban checks.
- `cookbooks/nginx-multisite/files/`: Static `index.html` files for the three sites and test/CI/status locations. Preserve as Ansible role files or replace with application-managed content.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource with backup behavior. It is not referenced by the reviewed entrypoint path; do not migrate it unless additional recipes use it.

### Target Details

- **Operating System**: The Vagrant box is Fedora 42, but the provisioning script uses Debian/Ubuntu commands (`apt-get`), and the cookbooks claim support for Ubuntu >=18.04 and CentOS >=7. Nginx paths, `www-data`, UFW, and `/var/log/auth.log` also suggest Debian-family assumptions. Resolve this contradiction before implementation. If no target is selected, standardize the Ansible baseline on Ubuntu 22.04/24.04 or explicitly build a Fedora/RHEL-compatible variant; do not assume the current Fedora box will run unchanged.
- **Virtual Machine Technology**: Vagrant with the libvirt provider; 2 vCPUs, 2 GB memory, private networking, and forwarded HTTP/HTTPS ports.
- **Cloud Platform**: Not specified. No AWS, Azure, or GCP integration was found.

## Migration Approach

### Key Dependencies to Address

- **Ansible controller and collections**: Define a supported Ansible version and use `ansible.posix`, `community.general`, and `community.postgresql` where appropriate. Pin collection versions in `requirements.yml`.
- **Nginx**: Replace Chef `nginx ~> 12.0` and custom templates with an Ansible Nginx role or a repository-owned role. Package names, service names, site-enable paths, and user/group names must be OS-specific.
- **Memcached (`~> 6.0`)**: Replace the Chef cookbook with distribution packages and an Ansible service/configuration role. Confirm whether Memcached should listen only on localhost and whether authentication or network restrictions are required.
- **Redis (`~> 7.2.4`)**: Replace `redisio` with a supported Ansible role or native package/configuration tasks. Preserve port and authentication behavior, but avoid shell-editing generated config and verify the intended Redis version and replication model.
- **PostgreSQL**: Replace package/service shell commands with `community.postgresql.postgresql_user` and `postgresql_db` (or equivalent), using a controlled admin connection and idempotent privilege management.
- **Python application**: Replace the unpinned `main` Git checkout and raw pip command with a pinned commit/tag, a virtualenv, and preferably a locked requirements file. Confirm the application’s expected runtime and migration/startup behavior.
- **systemd**: Replace the inline Chef unit with a Jinja2-managed unit and `ansible.builtin.systemd_service`, including daemon reload and health verification.
- **TLS certificates**: Development self-signed certificates can be generated with `community.crypto` or OpenSSL tasks. Production certificates should come from an approved PKI/ACME process and should not be generated with fixed subject data.
- **Firewall and fail2ban**: Replace UFW shell commands with `community.general.ufw` or an OS-native firewall role. Configure fail2ban using a supported package and jail template, taking OS-specific log paths into account.

### Security Considerations

- **Plaintext Redis credential**: `redis_secure_password_123` is embedded in `cookbooks/cache/recipes/default.rb`. Treat it as compromised, rotate it, and store the replacement in Ansible Vault or an external secret manager. Restrict Redis exposure and test authentication.
- **Plaintext PostgreSQL credentials**: The FastAPI recipe creates user `fastapi` with password `fastapi_password` and writes the same value into a world-readable (`0644`) `.env` file. Rotate it, store it in Vault, set restrictive file permissions, and avoid exposing it in command-line process output or logs.
- **TLS private keys and certificates**: Keys are placed under `/etc/ssl/private` with `root:ssl-cert` and mode `0640`; preserve least privilege. The current certificates are self-signed and valid for 365 days, with an email and example organization embedded in the subject. Define a production certificate lifecycle and renewal process.
- **TLS policy**: TLS 1.2/1.3, selected RSA/DHE/ECDHE ciphers, HSTS, and security headers are configured. Validate cipher compatibility, certificate algorithms, HSTS impact, and whether `X-XSS-Protection` remains required. Run automated TLS and header tests after migration.
- **SSH hardening**: Root login and password authentication are disabled by editing `sshd_config`. Ensure an alternate administrative key/user exists before applying this task, use a validated Ansible template/drop-in, and test with `sshd -t` before restart.
- **Firewall exposure**: UFW allows SSH, HTTP, and HTTPS, while the FastAPI service listens on `0.0.0.0:8000` and Redis/PostgreSQL defaults need explicit binding review. Permit only intended service paths; ideally keep application and data services private.
- **Nginx and fail2ban configuration**: Preserve rate limits, request-size limits, access restrictions for `.git`/`.svn` and `.ht*`, and fail2ban jails, but validate that the configured log paths and filters exist on the selected OS.
- **Supply-chain controls**: The Chef bootstrap downloads and executes `https://omnitruck.chef.io/install.sh`, and the application is cloned from an unpinned `main` branch. Replace with verified, pinned artifacts and document checksum/signature policy.
- **Privilege and process model**: The FastAPI systemd service runs as `root`; migrate to a dedicated unprivileged service account with only the required filesystem and network permissions unless application constraints prove otherwise.

### Technical Challenges

- **OS inconsistency**: Fedora 42/libvirt conflicts with Debian package commands, UFW assumptions, `www-data`, and auth log paths. Select and test a supported OS first, or implement OS-family variables and conditional roles.
- **Chef attribute precedence and defaults**: Site paths differ between `solo.json` (`/var/www/...`) and cookbook defaults (`/opt/server/...`). Establish the authoritative target values and preserve the three sites explicitly in Ansible group/host variables.
- **Idempotence and unsafe shell workarounds**: Redis config cleanup, PostgreSQL `|| true`, `sed` edits, and firewall shell commands can hide failures. Replace with modules, managed files/drop-ins, validation commands, and explicit failure handling.
- **Nginx template correctness**: The virtual-host template contains conditional server blocks and strict limits that may affect normal application requests. Render each site, run `nginx -t`, and perform HTTP/HTTPS smoke tests before enabling handlers.
- **Certificate bootstrapping**: Self-signed certificates make local testing easy but are unsuitable for production. Define separate development and production variables and certificate sources.
- **Application deployment lifecycle**: A Git checkout, pip install, database creation, and systemd start are coupled in one cookbook. Introduce clear handlers, readiness checks, database migrations if required, rollback strategy, and pinned dependencies.
- **Dependency and version drift**: Chef Supermarket constraints do not directly map to Ansible packages or roles. Create a compatibility matrix for Nginx, Redis, Memcached, PostgreSQL, Python, and the selected OS.
- **Environment reachability**: The documented host IP in `vagrant-provision.sh` (`192.168.56.10`) differs from the Vagrant private IP (`192.168.121.10`). Correct documentation and test DNS/hosts resolution for all virtual hosts.

### Migration Order

1. **Foundation and target contract**: Decide Ubuntu versus Fedora/RHEL, fix Vagrant networking/documentation, define inventories, Ansible version, collections, variable conventions, and secret storage.
2. **Nginx static-site baseline**: Migrate package/service installation, site directories, static files, virtual hosts, HTTP/HTTPS behavior, and configuration validation. This provides an early visible smoke-test target.
3. **Host security**: Add SSH hardening, firewall policy, sysctl settings, fail2ban, and Nginx security headers/rate limits after establishing safe administrative access. Test in a disposable VM.
4. **TLS lifecycle**: Implement development self-signed certificates first, then integrate the chosen production PKI/ACME process and renewal tests.
5. **Cache services**: Migrate Memcached and Redis, remove the Ruby workaround, rotate the Redis secret, restrict listeners, and verify service persistence and authentication.
6. **PostgreSQL and FastAPI application**: Provision the database securely, create a dedicated application user, deploy the pinned application revision and virtualenv, install the systemd unit under a non-root account, and add endpoint/health checks.
7. **Integration, rollback, and handover**: Run convergence twice, validate all three hostnames and forwarded ports, test reboot persistence, scan TLS/firewall exposure, document operations, and decommission Chef/Berkshelf only after acceptance.

### Assumptions

- The three local cookbooks are the complete migration scope; no additional Chef roles, environments, data bags, encrypted data bags, or production wrapper cookbooks were present in the supplied tree.
- `solo.json` is intended to override cookbook defaults, but the conflicting site document roots must be resolved with the application owner.
- The three cluster-local names are development/test names, and self-signed certificates are acceptable only for that environment.
- No cloud platform, external load balancer, DNS automation, monitoring, backup, or centralized secret manager is defined in the repository.
- The external Chef cookbooks may contain behavior not visible in this repository; their exact package versions and generated configuration must be validated during migration.
- PostgreSQL is intended to run locally on the same host as FastAPI, and Redis/Memcached are intended to be local services; this must be confirmed before choosing a multi-host Ansible inventory.
- The FastAPI repository’s requirements, database schema, health endpoint, and deployment contract were not audited; application owners must supply those requirements.
- The desired Ansible target is expected to retain equivalent service behavior, not necessarily reproduce Chef resource ordering or implementation details.
- Credentials found in the source should be considered test placeholders or compromised and must not be carried into the Ansible repository.
- The current Vagrant box and bootstrap script are not internally consistent; migration acceptance requires a selected OS and a clean reproducible build.
- The commented Chef SSL cookbook is not an active dependency and should not be migrated unless a production certificate workflow requires its functionality.
- A staged cutover is assumed: build and test Ansible alongside Chef, compare rendered configuration and service behavior, then remove Chef provisioning after sign-off.
