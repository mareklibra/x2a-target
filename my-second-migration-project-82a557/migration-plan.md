# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant environment containing three local cookbooks: a multi-site Nginx/security stack, caching services, and a FastAPI/PostgreSQL application. The migration is moderate in implementation risk rather than scale: there are only three cookbooks, but the repository combines OS provisioning, application deployment, databases, credentials, firewall/SSH hardening, TLS generation, and third-party cookbook dependencies. A focused team should be able to produce an initial Ansible implementation in approximately 1–2 weeks, with another 1–2 weeks for cross-platform validation, security remediation, idempotency testing, and operational handover. The timeline assumes the application repository and TLS ownership are clarified early.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web-server configuration for three SSL-enabled virtual hosts (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), including document roots, static landing pages, site enablement, and service lifecycle management.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Nginx installation and configuration, templated `nginx.conf` and security configuration, per-site virtual hosts, symlinks under `sites-enabled`, static site content, self-signed certificates for development, fail2ban, UFW, sysctl hardening, and SSH restrictions.
    - Migration shape: An Ansible role with separate tasks for packages, Nginx, sites, TLS, and host hardening; use Jinja2 templates, handlers, `ansible.builtin.ufw` or equivalent firewall tasks, and explicit certificate-management variables.

- **cache**:
    - Description: Installs and enables Memcached and Redis, configures a Redis server on port 6379 with password authentication, creates the Redis log directory, and applies a post-generation configuration workaround.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: External Memcached and Redis cookbook usage, Redis password configuration, service enablement, log directory creation, and removal of selected replication-related Redis settings through a Ruby block.
    - Migration shape: An Ansible role using distribution packages or supported vendor repositories, managed Redis/Memcached configuration, systemd handlers, and a documented replacement for the current workaround. Redis credentials must be moved to Ansible Vault or an external secret manager.

- **fastapi-tutorial**:
    - Description: Deploys the `dibanez/fastapi_tutorial` Git repository, installs Python and PostgreSQL prerequisites, creates a Python virtual environment, installs requirements, initializes a PostgreSQL database/user, writes application environment settings, and runs Uvicorn through systemd.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Git-based application deployment, Python virtualenv, pip dependency installation, PostgreSQL service management, database/user bootstrap, `.env` generation, and a root-running systemd service listening on port 8000.
    - Migration shape: An Ansible role or application playbook with package, Git, Python, PostgreSQL, database, systemd, and secret tasks. Replace shell-based SQL commands with PostgreSQL Ansible modules and make schema/database creation genuinely idempotent.

**CRITICAL PATH VERIFICATION:**
The three cookbook paths above are present in the supplied repository tree. No Puppet modules, Salt states, or PowerShell modules were identified. The `recipes/` files are Chef Ruby recipes, and each cookbook has a `metadata.rb` entrypoint/metadata file.

### Infrastructure Files

- `Berksfile`: Chef Supermarket and local cookbook dependency declarations. During migration it becomes an inventory of external behavior to replace, not a runtime dependency file.
- `solo.rb`: Chef Solo configuration, including cookbook search paths, cache path, and logging. Replace with Ansible inventory, `ansible.cfg`, playbooks, and role paths.
- `solo.json`: Chef run list and node attributes. Convert the run list into play/role ordering and migrate site, TLS, security, and service variables into group/host vars with protected secrets in Vault.
- `Vagrantfile`: Defines a `generic/fedora42` VM, libvirt resources, private/forwarded networking, and rsync of the repository to `/chef-repo`. It should be retained or revised as an Ansible test harness; the guest OS and provisioner assumptions require correction.
- `vagrant-provision.sh`: Installs Chef, Berkshelf, dependencies, and runs Chef Solo. Replace with an Ansible provisioner or a documented `ansible_local`/host-run workflow. Do not retain unaudited remote installer behavior in production provisioning.
- `cookbooks/nginx-multisite/attributes/default.rb`: Default virtual-host, certificate-path, and security settings. Convert to role defaults, with environment-specific values in inventory/group variables.
- `cookbooks/nginx-multisite/templates/*.erb`: Nginx, virtual-host, fail2ban, and sysctl configuration templates. Convert to Jinja2 templates and handlers.
- `cookbooks/nginx-multisite/files/default/*/index.html`: Static test/CI/status page content. Copy into an Ansible role's `files/` tree or, preferably, deploy application-owned content separately.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource; inspect its actual usage during implementation and replace with native Ansible `lineinfile`, `blockinfile`, or template tasks.

### Target Details

- **Operating System**: The Vagrant box is explicitly `generic/fedora42`, but `vagrant-provision.sh` runs `apt-get`, and the recipes use Debian-oriented paths/users such as `www-data`, UFW, and the `ssh` service name. Cookbook metadata claims Ubuntu `>=18.04` and CentOS `>=7.0`. The effective target is therefore unresolved. Validate Fedora 42 versus Ubuntu/Debian before role design; use OS-family conditionals only if both are required. Do not assume the current provisioning works on Fedora.
- **Virtual Machine Technology**: Vagrant with the libvirt provider, 2 CPUs, 2048 MB RAM, a private address of `192.168.121.10`, and ports 80/443 forwarded to host ports 8080/8443.
- **Cloud Platform**: Not specified. No AWS, Azure, or GCP configuration was found.

## Migration Approach

### Key Dependencies to Address

- **Chef Solo/Berkshelf**: Replace `solo.rb`, `solo.json`, and `berks install/vendor` with an Ansible project layout, inventory, `ansible.cfg`, roles, collections requirements, and CI/linting.
- **`nginx` cookbook `~> 12.0`**: The local cookbook currently installs Nginx directly rather than clearly including the external Nginx recipe. Reproduce only the required package/service/configuration behavior with Ansible `apt`/`dnf`, `template`, `file`, and handlers; verify the effective Chef dependency graph before retiring it.
- **`memcached` cookbook `~> 6.0`**: Replace with platform package installation and an explicit Memcached configuration/service role. Confirm whether Memcached is actually consumed by the application.
- **`redisio` cookbook `~> 7.2.4`**: Replace with a supported Redis package/repository and managed configuration. Preserve only required settings and document the reason for removing replication directives.
- **Python and application requirements**: The FastAPI repository's `requirements.txt` is fetched at deployment time. Pin the application revision and dependency lock material, use a virtualenv, and decide whether the application should be built from a release artifact rather than mutable `main`.
- **PostgreSQL**: Use `community.postgresql` modules and a proper authentication policy. Ensure the target PostgreSQL package/version and socket authentication behavior are defined.
- **Systemd**: Convert the inline unit to a version-controlled Jinja2 unit template with `daemon_reload` and restart handlers; define a non-root service account.
- **TLS and security tools**: Replace OpenSSL shell commands with either managed development certificates or a production certificate workflow (ACME, PKI, or supplied certificates). Replace UFW/fail2ban shell logic with supported Ansible modules/roles and validated templates.

### Security Considerations

- **Hardcoded Redis credential**: `redis_secure_password_123` is embedded in `cookbooks/cache/recipes/default.rb`. Rotate it, store it in Ansible Vault or an external secret manager, and avoid writing it into world-readable configuration or logs.
- **Hardcoded PostgreSQL credentials**: The FastAPI recipe creates user `fastapi` with password `fastapi_password` and writes the same password into `/opt/fastapi-tutorial/.env` with mode `0644`. Treat it as compromised: rotate it, use Vault, set restrictive file permissions, and use PostgreSQL modules rather than interpolated shell SQL.
- **TLS private keys**: Self-signed keys are generated under `/etc/ssl/private` with `root:ssl-cert` and mode 0640. Preserve restrictive ownership, but separate development certificates from production PKI and ensure private keys are never committed or exposed through logs. The current generated certificate subject uses example organization values and 365-day validity.
- **SSH hardening**: Root login and password authentication are disabled through `sed`. Reimplement with validated `sshd_config` management, ensure a working key-based administrative account exists first, and test connectivity before applying changes.
- **Firewall exposure**: UFW defaults to deny and allows SSH, HTTP, and HTTPS. Reconcile this with the Vagrant forwarded ports and the FastAPI port 8000; do not expose 8000 unless explicitly required. Apply firewall changes in a safe order to avoid lockout.
- **Fail2ban/sysctl**: Migrate the jail and kernel settings from templates, validate settings against the selected OS, and test that fail2ban does not block automation or health checks.
- **Application execution**: The systemd service runs Uvicorn as `root`, and the application directory is root-owned. Create a dedicated least-privilege account, define writable directories explicitly, and constrain systemd permissions where practical.
- **Supply-chain risks**: Chef installation uses a remote `curl` installer, the application tracks Git `main`, and pip requirements are installed dynamically. Pin versions/revisions, verify checksums or signatures where possible, and use internal mirrors/artifacts for production.
- **Credential inventory by module**: `cache` contains a Redis password; `fastapi-tutorial` contains PostgreSQL credentials and a database URL; `nginx-multisite` generates private TLS keys but has no visible external secret store. No Chef encrypted data bags or Chef Vault usage were identified in the reviewed files.

### Technical Challenges

- **OS mismatch**: Fedora 42 is declared while the shell provisioner uses `apt-get` and the recipes assume Debian paths, UFW, and `www-data`. Select and test the supported OS first; add explicit `ansible_os_family` handling only when justified.
- **Chef behavior is split across recipes, attributes, templates, and external cookbooks**: Build role boundaries around Nginx, security, cache, and FastAPI rather than performing a line-by-line Ruby translation. Use handlers and check-mode/idempotency tests.
- **Configuration source mismatch**: `nginx-multisite/attributes/default.rb` uses `/opt/server/{test,ci,status}`, while `solo.json` overrides roots to `/var/www/{...}`. The migration must establish one authoritative variable set and verify which paths are intended.
- **Nginx/site portability**: The templates and ownership likely assume Debian Nginx layout. Validate Fedora versus Debian package layout, service names, include paths, and the `www-data` account.
- **TLS lifecycle**: Current certificates are self-signed and generated only if files are absent. Define renewal, replacement, SANs, trust distribution, and production certificate ownership before implementing the role.
- **Redis workaround**: The Ruby block removes several generated directives after `redisio` runs. Determine the Redis version and intended replication model; replace the workaround with a supported, declarative configuration rather than preserving fragile text manipulation.
- **Database/application idempotency**: Shell commands use `|| true`, masking failures and making drift hard to detect. Use module-based database creation, migrations, health checks, and an explicit deployment/restart policy.
- **External application mutability**: Cloning `main` means repeated runs can change the deployed application. Pin a commit or release and coordinate application ownership with the Ansible team.
- **Testing and connectivity**: The Vagrant script prints `192.168.56.10`, while the Vagrant private IP is `192.168.121.10`. Correct validation URLs and add smoke tests for all three hostnames, HTTP/HTTPS, Redis, PostgreSQL, and the FastAPI endpoint.

### Migration Order

1. **Foundation and test harness**: Resolve the target OS, correct Vagrant networking/provisioning, establish Ansible inventory, Vault policy, linting, Molecule or equivalent tests, and common package/service conventions.
2. **nginx-multisite base and static sites**: Migrate Nginx installation, templates, document roots, site links, and handlers. This provides visible smoke-test value and establishes the reverse-proxy/web-server pattern.
3. **Security controls**: Add SSH hardening, firewall, fail2ban, sysctl, ownership, and TLS handling after a reliable administrative access path exists. Validate in disposable VMs before production use.
4. **cache**: Migrate Memcached and Redis independently, introduce secret management, and validate clients before removing Chef dependencies. Resolve the Redis configuration workaround during this stage.
5. **fastapi-tutorial and PostgreSQL**: Migrate the application, database, environment secrets, systemd unit, and health checks. Pin the Git revision and use a dedicated service identity.
6. **Cutover and decommissioning**: Run Chef and Ansible in parallel on disposable/staging hosts, compare effective configuration, perform rollback testing, then retire Berkshelf/Chef provisioning and document ownership.

### Assumptions

- The repository is the complete relevant Chef source; no hidden roles, data bags, encrypted secrets, CI pipelines, or production certificate files were supplied.
- The three directories under `cookbooks/` are the complete local cookbook inventory.
- `solo.json` is intended to override the Nginx site paths and security settings, although the precedence and deployed state should be verified.
- The external cookbook versions in `Berksfile` are intended dependencies even where the local recipes use only some of their behavior.
- The target operating system is currently ambiguous: Fedora 42 is declared by Vagrant, while the provisioning and cookbook implementation are primarily Debian/Ubuntu-oriented.
- Development certificates are acceptable only for local testing; production TLS requirements, certificate authority, renewal, and secret distribution are not defined.
- The FastAPI Git repository, its `main` branch, package requirements, port 8000 behavior, and PostgreSQL schema are externally controlled and require application-team confirmation.
- The intended production topology is not specified: it is unclear whether PostgreSQL, Redis, Memcached, Nginx, and FastAPI will remain on one host or be separated.
- No high-availability, backup, monitoring, log shipping, DNS, load-balancer, or cloud deployment requirements are visible.
- The application may not need Memcached, despite the cache cookbook installing it; usage must be confirmed before migrating and operating it.
- The current credentials are examples but must nevertheless be rotated because they are present in source code.
- The migration should preserve externally observable behavior while improving idempotency, least privilege, secret handling, version pinning, and platform support.
- Team coordination should assign separate owners for platform/OS, Nginx/security, data services, and FastAPI deployment, with a shared inventory/variable contract and staging acceptance checklist.
