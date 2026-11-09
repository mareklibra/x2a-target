# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo infrastructure repository containing three local cookbooks and a Vagrant/libvirt development environment. The migration scope is approximately three Ansible roles, one coordinating playbook, supporting templates/files, and replacement of Chef/Berkshelf dependency handling. The effort is moderate rather than large, but security controls, application/database provisioning, and conflicting platform assumptions require validation before production use. A realistic estimate is 2–4 weeks for one engineer, including implementation, testing, security review, and handover; parallel work by application and platform owners can reduce elapsed time.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx reverse proxy/web server configuration for three SSL-enabled virtual hosts: `test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`. It installs Nginx, creates site roots and sample index pages, configures virtual hosts, generates development certificates, and applies host security hardening.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Nginx service management, templated `nginx.conf` and virtual hosts, symlinked enabled sites, self-signed RSA certificates, Fail2ban, UFW default-deny firewall rules, sysctl security settings, root-login and SSH password-authentication hardening

- **cache**:
    - Description: Local cache-service cookbook that installs and enables Memcached and Redis, creates Redis logging storage, configures Redis on port 6379 with authentication, and removes selected replication-related directives using a post-configuration workaround.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: Memcached inclusion, RedisIO configuration, Redis service enablement, Redis log directory, compatibility workaround implemented with a Ruby block

- **fastapi-tutorial**:
    - Description: Development application stack for a FastAPI tutorial. It installs Python and PostgreSQL packages, clones the application from GitHub, creates a virtual environment, installs application requirements, creates a PostgreSQL database/user, writes a `.env` file, and runs Uvicorn through systemd.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Python 3 virtualenv, Git-based application deployment, PostgreSQL service/database setup, systemd unit, Uvicorn on `0.0.0.0:8000`

**CRITICAL PATH VERIFICATION:**
All module paths above were present in the supplied repository tree, and each cookbook’s `recipes/default.rb` entrypoint was reviewed. No Puppet modules, Salt states, or PowerShell modules were identified.

### Infrastructure Files

- `Berksfile`: Defines local cookbook sources and external Chef Supermarket dependencies. Ansible will replace this dependency graph with roles, collections, and pinned package/application inputs.
- `Vagrantfile`: Defines a Fedora 42 Vagrant box, hostname `chef-nginx`, private IP `192.168.121.10`, HTTP/HTTPS forwarded ports, a libvirt provider with 2 GB RAM and 2 CPUs, rsync of the repository, and shell provisioning.
- `vagrant-provision.sh`: Installs system build tools, Chef, Berkshelf, cookbook dependencies, and runs Chef Solo. It is the principal bootstrap path to replace with Ansible installation/invocation or an Ansible-compatible provisioner.
- `solo.rb`: Chef Solo cache, cookbook search paths, and logging configuration. It has no direct Ansible equivalent beyond inventory, role paths, and logging conventions.
- `solo.json`: Chef run list and node attributes. It supplies the active Nginx sites, SSL paths, and security settings; it also overrides cookbook defaults for site document roots.
- `cookbooks/nginx-multisite/attributes/default.rb`: Cookbook defaults for sites, certificate locations, Fail2ban/UFW enablement, and SSH hardening.
- `cookbooks/nginx-multisite/templates/`: Nginx, virtual-host, security, Fail2ban, and sysctl templates to translate into Ansible-managed Jinja2 templates.
- `cookbooks/nginx-multisite/files/`: Static sample `index.html` content for the three sites, to be moved into an Ansible role’s `files/` tree or replaced by application deployment content.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource; inspect and either replace with `ansible.builtin.lineinfile`/`blockinfile` or remove if unused by the reviewed default path.

### Target Details

- **Operating System**: The Vagrant box is explicitly Fedora 42, but the provisioning script uses `apt-get`, and the cookbooks declare support for Ubuntu `>= 18.04` and CentOS `>= 7.0`. The Nginx recipe uses Debian-style `www-data` ownership and UFW, while the target platform is Fedora. OS compatibility is therefore unresolved. Select and test a supported target, preferably an explicitly defined RHEL-family or Ubuntu release, and use OS-family-specific package, service, firewall, web-root, and SSH tasks.
- **Virtual Machine Technology**: Vagrant using the `libvirt` provider. The VM is configured with 2 CPUs, 2048 MB RAM, rsync folder synchronization, and private/forwarded networking.
- **Cloud Platform**: Not specified. No AWS, Azure, GCP, or cloud-init configuration was found.

## Migration Approach

Create an Ansible project with `roles/nginx_multisite`, `roles/cache`, and `roles/fastapi_tutorial`, plus a site playbook equivalent to the Chef run list. Put site definitions, ports, certificate locations, security switches, Redis settings, and application/database credentials in group/host variables, with secrets encrypted by Ansible Vault or an external secret manager. Preserve handler-driven service reload/restart behavior and use check mode/idempotence tests before cutover.

Translate Chef templates to Jinja2 templates and static files to role files. Replace Chef resources with native Ansible modules wherever possible: `package`, `service`, `systemd`, `template`, `copy`, `file`, `user`, `community.postgresql.postgresql_db`, `community.postgresql.postgresql_user`, `ansible.posix.firewalld` or `community.general.ufw`, `ansible.posix.sysctl`, and `ansible.builtin.git`. Avoid shell commands for database creation, firewall configuration, and SSH editing unless a module cannot express the required behavior.

### Key Dependencies to Address

- **Nginx Supermarket cookbook `nginx ~> 12.0`**: Replace with an Ansible Nginx role or a local role using distribution packages and reviewed templates. Pin the OS package repository/version policy.
- **Memcached cookbook `~> 6.0`**: Replace with Ansible package/service tasks and an explicit Memcached configuration; verify whether Memcached is actually required by the application.
- **RedisIO cookbook `~> 7.2.4` / `redisio` metadata dependency**: Replace with a supported Redis package or approved Redis role. Model authentication, bind addresses, persistence, and service configuration explicitly rather than relying on Chef node attributes.
- **PostgreSQL system packages and local commands**: Use the `community.postgresql` collection and PostgreSQL service tasks. Define database/user ownership and privilege behavior idempotently.
- **Python/pip dependencies from the cloned repository**: Keep the Git revision pinned to a release or commit rather than `main`; use a virtualenv and preferably a lock/constraints file. Decide whether application build dependencies belong in this infrastructure migration.
- **Chef Solo/Berkshelf/Omnibus Chef**: Remove from the runtime path. Use `ansible-core`, a pinned collection requirements file, an inventory, and CI lint/test tooling.
- **Vagrant/libvirt**: Retain initially as a local test harness, but replace shell Chef provisioning with an Ansible provisioner or a script invoking `ansible-playbook`.

### Security Considerations

- **Hardcoded Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Move it to Ansible Vault or an external secret store, rotate it, and avoid exposing it in logs, facts, rendered files, or command lines.
- **Hardcoded PostgreSQL credential**: `fastapi-tutorial` creates user `fastapi` with password `fastapi_password` and writes the same credential into `/opt/fastapi-tutorial/.env` with mode `0644`. Treat it as compromised, rotate it, store the `.env` with restrictive permissions (normally `0600`), and use Vault-backed variables.
- **TLS certificates and private keys**: The Nginx role generates self-signed certificates under `/etc/ssl/certs` and private keys under `/etc/ssl/private`. This is explicitly development-oriented and causes browser warnings. Production migration must define certificate issuance/import, renewal, SANs, key ownership, backup, and rotation. Never commit private keys.
- **SSH hardening**: Root login and password authentication are disabled using `sed` and service restart handlers. Replace with validated `sshd_config` management, preserve an authorized administrative key path, run `sshd -t` before restart, and test console/recovery access to avoid lockout.
- **Firewall and intrusion prevention**: UFW is configured to deny by default and allow SSH/HTTP/HTTPS, while the Fedora target normally favors firewalld. Select one platform-appropriate implementation and explicitly account for forwarded ports and the FastAPI port 8000. Validate Fail2ban jails and log paths on the selected OS.
- **Privilege and service identity**: The FastAPI systemd service runs as `root`, and the cloned application is initially owned by root. Create a dedicated least-privilege service account, restrict application and environment-file permissions, and define writable directories separately.
- **Supply-chain controls**: The application is cloned from GitHub at `main`, and Chef dependencies are resolved from Supermarket without all versions being pinned in the Berksfile. Pin Git revisions, package repositories, collection versions, and role sources; scan dependencies and validate checksums where feasible.
- **Per-module credential patterns**: `cache` contains a Redis password; `fastapi-tutorial` contains PostgreSQL and application database credentials; `nginx-multisite` references private-key paths and generates keys but contains no imported production certificate material. No encrypted data bags or Chef Vault usage were visible in the reviewed files.

### Technical Challenges

- **OS inconsistency is the principal blocker**: Fedora 42 is provisioned with `apt-get`, UFW, Debian ownership conventions, and Ubuntu/CentOS cookbook support. Establish the target OS before implementation and test package names, service names, Nginx layout, firewall, PostgreSQL, and SSH behavior on it.
- **Conflicting Nginx site roots**: `nginx-multisite` defaults to `/opt/server/...`, while `solo.json` overrides them to `/var/www/...`; the static files are placed according to the runtime attributes. Preserve the intended JSON override in Ansible variables and make the desired paths explicit.
- **TLS is development-only**: Self-signed certificates are generated with shell commands and a one-year lifetime. Production certificate management cannot be inferred from this repository and needs an owner, certificate source, renewal process, and validation plan.
- **Chef ordering and implicit dependency behavior**: The run list applies Nginx, cache, and FastAPI together, while each cookbook includes external recipes. Ansible roles should define explicit ordering and handlers; PostgreSQL must be available before database creation, and Nginx must not reload until certificates and virtual-host files are valid.
- **Non-idempotent or fragile shell workarounds**: Redis configuration is modified with a Ruby block, database creation uses `psql` plus `|| true`, UFW uses shell status checks, and SSH uses `sed`. Replace these with idempotent modules and convergence tests.
- **Application deployment ambiguity**: The service runs as root from the `main` branch and assumes `app.main:app`, port 8000, and a requirements file. Confirm release strategy, migrations, health checks, process ownership, and whether Nginx should proxy to FastAPI.
- **Template behavior and custom resource coverage**: Nginx, Fail2ban, sysctl, and site templates were not exhaustively audited for all directives. Validate rendered configuration with `nginx -t`, `fail2ban-client`, and `sysctl` checks. Confirm whether the custom `lineinfile` resource is used outside the default path.
- **Testing/network assumptions**: The Vagrantfile uses `192.168.121.10`, while the provisioning output references `192.168.56.10`; forwarded ports are 8080/8443. Correct documentation and add automated smoke tests for DNS/hosts resolution, TLS, HTTP status, SSH access, Redis, PostgreSQL, and FastAPI.

### Migration Order

1. **Foundation and target-platform decision**: Agree on OS, network/DNS model, certificate strategy, secret manager, Ansible and collection versions, inventory structure, and Vagrant test approach.
2. **nginx-multisite**: Migrate the web/server baseline, static sites, templates, TLS abstraction, firewall, Fail2ban, sysctl, and SSH hardening. This is the highest operational/security impact and provides the primary integration test target.
3. **cache**: Migrate Memcached and Redis with secret rotation, explicit bind/auth/persistence settings, and service validation. Resolve the Redis workaround against the chosen Redis version rather than carrying it forward blindly.
4. **fastapi-tutorial**: Migrate packages, pinned application checkout, virtualenv, PostgreSQL database/user, secure environment configuration, dedicated service account, systemd unit, and application health checks.
5. **Integration and cutover**: Run the full playbook repeatedly in Vagrant, compare rendered configuration and service behavior with Chef, perform security review, document rollback, then disable Chef/Berkshelf provisioning.

### Assumptions

- The three directories under `cookbooks/` are the complete local cookbook inventory.
- `solo.json` is the effective intended node configuration and its Nginx document-root values take precedence over cookbook defaults.
- The repository represents a development/tutorial environment; production certificate issuance, DNS, monitoring, backups, and HA are not defined.
- Fedora 42 is the current VM test target, but the existing provisioning script is not compatible with it as written; the migration must not assume Fedora works without correction.
- No external Chef cookbook source code was reviewed; behavior of `nginx`, `memcached`, and `redisio` must be validated or replaced by explicit Ansible implementation.
- The application repository, its requirements, schema migrations, and runtime behavior are outside this repository and require application-owner confirmation.
- The FastAPI service is intentionally exposed on all interfaces at port 8000, although the firewall currently documents only SSH/HTTP/HTTPS; desired exposure must be confirmed.
- Self-signed certificates are acceptable only for local testing; production TLS requirements are a separate decision.
- Secrets shown in source are examples but must nevertheless be rotated and removed from plaintext configuration during migration.
- The unused/commented Chef provisioner path and the `lineinfile` custom resource may not be part of the active runtime path; confirm before deleting them.
- A two-to-four-week estimate assumes one small team, prompt access to platform/application owners, and no requirement to reproduce undocumented external cookbook behavior exactly.
- Team coordination should assign platform ownership for OS/firewall/SSH/TLS, application ownership for FastAPI/PostgreSQL/runtime requirements, and security ownership for secrets, hardening, and certificate review. Maintain a migration matrix, test evidence, decision log, and rollback runbook in version control.
