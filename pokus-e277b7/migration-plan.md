# MIGRATION FROM CHEF TO ANSIBLE

This is a small Chef Solo repository with three local cookbooks and a Vagrant/libvirt development environment. The migration scope is moderate rather than large: the desired state is concentrated in Nginx virtual hosts, host security, caching services, and a FastAPI/PostgreSQL demo application. The main risks are plaintext credentials, shell-driven configuration, self-signed certificate handling, platform inconsistencies, and unclear dependency behavior in the external cookbooks. A focused team could produce an initial Ansible implementation in approximately 2–3 weeks, including validation on the Vagrant target; production hardening and application/service acceptance testing may require an additional 1–2 weeks.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Installs and runs Nginx for three configured sites (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), creates site document roots and static index pages, enables SSL virtual hosts, and applies host and web-server security settings.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Nginx service management, generated per-site configuration, `sites-available`/`sites-enabled` links, self-signed RSA certificates, Fail2ban, UFW, sysctl hardening, disabled root SSH login, disabled SSH password authentication, and packaged static content.
    - Ansible mapping: A role with separate tasks for Nginx, sites, TLS, and security; Jinja2 templates for the reviewed ERB files; handlers for Nginx, Fail2ban, SSH, and sysctl changes; variables for site definitions and certificate paths.

- **cache**:
    - Description: Installs and enables Memcached and Redis, configures one Redis server on port 6379 with password authentication, creates the Redis log directory, and performs a post-configuration cleanup of replication-related Redis directives.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: External `memcached` and `redisio` cookbook integration, Redis authentication, service enablement, log directory ownership, and a Ruby-based configuration workaround.
    - Ansible mapping: A cache role using distribution packages or explicitly selected Redis/Memcached roles/collections, native Redis configuration templates, systemd handlers, and an idempotent configuration model instead of text mutation.

- **fastapi-tutorial**:
    - Description: Installs Python 3 tooling, Git, PostgreSQL packages, and `libpq-dev`; checks out the FastAPI tutorial application; creates a virtual environment; installs its requirements; creates a PostgreSQL database and user; writes an application `.env`; and runs Uvicorn as a systemd service on port 8000.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Git checkout from `https://github.com/dibanez/fastapi_tutorial.git` at `main`, Python virtualenv, PostgreSQL bootstrap, systemd unit, automatic restart, and application environment configuration.
    - Ansible mapping: An application role using package, git, `ansible.builtin.pip`, `community.postgresql` database/user modules, a systemd unit template, and a vaulted application configuration. Pin the source revision and Python dependencies before treating it as a repeatable deployment.

**CRITICAL PATH VERIFICATION:**
All module paths above were present in the supplied repository tree and their primary entrypoints were reviewed. No Puppet, Salt, or PowerShell modules were identified.

### Infrastructure Files

- `Berksfile`: Chef Supermarket source, local cookbook declarations, and external dependencies. Replace dependency resolution with Ansible roles/collections and an explicit `collections/requirements.yml` and/or role requirements file.
- `Vagrantfile`: Defines the development target as `generic/fedora42` under libvirt with 2 GiB RAM, 2 CPUs, private IP `192.168.121.10`, and forwarded ports 8080/8443. Replace the Chef provisioner with `ansible_local` or an Ansible provisioner and make the target platform assumptions consistent.
- `vagrant-provision.sh`: Installs packages, Chef, Berkshelf, vendors cookbooks, and runs Chef Solo. Replace with Ansible installation/bootstrap only where needed; avoid piping the remote Chef installer pattern into the new workflow.
- `solo.rb`: Chef Solo cache, cookbook search paths, and logging configuration. Its concerns become Ansible inventory, project layout, fact gathering, and logging/verbosity settings.
- `solo.json`: Chef run list and runtime data. Convert the run list to a playbook/role sequence and migrate Nginx sites, TLS paths, and security variables into inventory/group/host vars, with secrets in Ansible Vault.
- `cookbooks/nginx-multisite/templates/`: ERB templates for Nginx, site, security, Fail2ban, and sysctl configuration. Convert to Jinja2 after validating syntax and platform-specific directives.
- `cookbooks/nginx-multisite/files/`: Static pages for the three sites. Move into role `files/` or deploy application-owned content separately.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom resource exists in the tree but was not referenced by the reviewed entrypoint recipes. Investigate before deleting or replacing it.

### Target Details

- **Operating System**: The Vagrant box is explicitly Fedora 42, but the provisioning script uses `apt-get`, the cookbooks support Ubuntu >=18.04 and CentOS >=7, and recipes use Debian-oriented names such as `www-data`, `ufw`, and the `ssh` service. The effective target is therefore ambiguous. Define supported distributions explicitly; either standardize on Fedora/RHEL-family behavior or change the development box to Ubuntu. Do not assume that Fedora 42 accepts the current package/service names.
- **Virtual Machine Technology**: Vagrant using the libvirt provider; 2 CPUs and 2048 MB memory. The synced repository path is `/chef-repo`.
- **Cloud Platform**: Not specified. No cloud-provider configuration was found.
- **Network/application endpoints**: The intended private address is `192.168.121.10`, with host forwarding for HTTP and HTTPS. The script also prints `192.168.56.10`, which conflicts with the Vagrant network declaration and should be corrected during migration.

## Migration Approach

### Key Dependencies to Address

- **Chef Supermarket `nginx` `~> 12.0`**: Determine which cookbook behavior is actually relied upon; replace with an Ansible Nginx role or `ansible.builtin.package` plus maintained templates. Avoid carrying over unused dependency behavior.
- **`memcached` cookbook `~> 6.0`**: Replace with distribution packages and an Ansible cache role; define listen address, service state, and resource limits explicitly.
- **`redisio` cookbook `~> 7.2.4` (Berksfile constraint is `~> 7.2.4`, while `cache/metadata.rb` only declares `redisio`)**: Replace with a supported Redis package/configuration role. Preserve authentication and service enablement, but redesign the Ruby post-edit workaround as a complete, version-aware config template.
- **Local cookbook run list**: `nginx-multisite`, `cache`, and `fastapi-tutorial` become ordered Ansible roles. Their current order should be reviewed rather than copied mechanically; application/database readiness and reverse-proxy routing need explicit dependencies.
- **FastAPI Git repository and Python requirements**: Pin a commit or release instead of tracking `main`; review `requirements.txt` and use a controlled Python package source/lock strategy.
- **PostgreSQL**: Use `community.postgresql` modules and a supported PostgreSQL role/package policy. Define authentication, locale, version, and idempotent migration behavior.
- **Vagrant/libvirt**: Use an Ansible-compatible Vagrant provisioner and test both the chosen OS and the network/forwarded-port model.

### Security Considerations

- **Plaintext Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Move it immediately to Ansible Vault or an external secret manager, avoid exposing it in task output, and rotate it because it has been stored in source.
- **Plaintext PostgreSQL credentials**: The FastAPI cookbook embeds `fastapi_password` in SQL commands and `/opt/fastapi-tutorial/.env`, with mode `0644`; the database URL repeats the password. Rotate it, store it in Vault, write the file with restrictive permissions (for example 0600), and use PostgreSQL modules rather than shell SQL with `|| true`.
- **TLS private keys**: Per-site keys are generated under `/etc/ssl/private` and assigned `root:ssl-cert` with mode 0640. Preserve least privilege, ensure Nginx can read keys, and decide whether development self-signed certificates should be replaced by an internal CA or ACME-managed certificates. Do not put private keys in Git or ordinary variables.
- **Self-signed certificate generation**: Certificates are generated with a one-year lifetime and a fixed example subject. Make certificate ownership, renewal, SANs, and trust expectations explicit; avoid shell interpolation of untrusted site names.
- **SSH hardening**: Root login and password authentication are disabled. Validate that inventory access uses a non-root administrative account with sudo and an installed SSH key before applying the changes, or the play may lock out operators.
- **Firewall and service exposure**: UFW denies by default and permits SSH, HTTP, and HTTPS. Preserve an emergency access path and validate whether Uvicorn port 8000, Redis 6379, Memcached, and PostgreSQL should remain localhost-only. Prefer Ansible firewall modules over shell commands.
- **Fail2ban and sysctl**: Migrate the supplied templates with platform-specific validation and handlers. Confirm jails match the actual Nginx/SSH logs and that sysctl values are approved by the security baseline.
- **Application privilege**: The systemd service runs the FastAPI application as `root`. Change to a dedicated unprivileged service account, with ownership and writable paths designed accordingly.
- **Supply-chain controls**: Chef and Berkshelf are installed dynamically, the Git application tracks `main`, and the provisioning script downloads an installer over curl. Pin Ansible collections, package repositories, application revisions, and trusted artifact sources.

### Technical Challenges

- **Platform mismatch**: Fedora 42 in Vagrant conflicts with `apt-get`, Debian package names, UFW assumptions, `www-data`, and potentially service names. Resolve the OS decision first and use `ansible_facts.os_family` only where cross-platform support is intentional.
- **Chef attribute precedence and path overrides**: Cookbook defaults use `/opt/server/{test,ci,status}`, while `solo.json` overrides them to `/var/www/{...}`. Treat the JSON values as the current intended runtime values and document variable precedence in Ansible.
- **Template and configuration portability**: ERB Nginx, Fail2ban, and sysctl templates were not fully audited here. Port them selectively, validate with `nginx -t`, `fail2ban-client`, and `sysctl --system`, and preserve handler ordering.
- **Non-idempotent shell behavior**: PostgreSQL creation uses shell commands and `|| true`; Redis edits a generated file with regex deletion; UFW uses command probes. Replace these with native modules and complete templates, with check-mode and repeat-run tests.
- **Nginx/application integration**: The reviewed Nginx recipes configure static sites, while FastAPI listens on port 8000 and no reviewed recipe shows an upstream/proxy relationship. Confirm whether Nginx should proxy the FastAPI service or only serve the three static sites.
- **Certificate/site assumptions**: Three SSL-enabled sites require matching certificate files and DNS/hosts resolution. Define whether certificates are pre-provisioned, generated in development, or issued by a certificate authority.
- **External cookbook behavior**: The exact versions resolved by Berkshelf and their platform support are not captured in a lockfile shown here. Reproduce the current dependency graph before replacing it.
- **Application reproducibility**: Tracking a Git branch and installing an unconstrained requirements file can change behavior independently of infrastructure. Establish an application release contract before migration acceptance.

### Migration Order

1. **Foundation and inventory**: Decide Ubuntu versus Fedora/RHEL, correct Vagrant networking/documentation, create Ansible project structure, pin collections, and establish Vault and CI/check-mode testing.
2. **nginx-multisite baseline**: Migrate packages, directories, static files, Nginx templates, handlers, and the three site variables. Validate HTTP/HTTPS, configuration tests, certificate permissions, and firewall access.
3. **Security controls**: Migrate SSH, UFW/firewall, Fail2ban, and sysctl with a preflight access check. Apply after basic connectivity is proven, or gate lockout-prone tasks.
4. **cache**: Migrate Redis and Memcached with vaulted Redis authentication, localhost/network restrictions, service tests, and a replacement for the Redis configuration workaround.
5. **fastapi-tutorial**: Pin the application revision, provision PostgreSQL idempotently, create a dedicated service user, deploy the virtualenv and systemd unit, and decide the Nginx reverse-proxy contract.
6. **Cutover and decommissioning**: Run repeated convergence and rollback tests, compare rendered configs and service states with Chef, update operator documentation, then retire Chef/Berkshelf/bootstrap scripts.

### Assumptions

- The three directories containing `metadata.rb` and `recipes/default.rb` are the complete local cookbook inventory.
- `solo.json` is the effective Chef Solo run configuration and overrides the Nginx default document-root values.
- The three external Berkshelf dependencies are intended to be used, although the precise resolved versions and generated vendor tree are not present in the supplied tree.
- The commented `ssl_certificate` dependency is not active and is not included as a required migration dependency.
- The intended deployment is primarily development/test because Vagrant, example domains, and self-signed certificates are used; production requirements are not specified.
- The source repository does not show DNS management, certificate authority integration, database migrations, application deployment lifecycle, monitoring, backups, or HA requirements.
- The FastAPI application repository, its `requirements.txt`, and all reviewed ERB templates were not opened because the plan is a high-level survey; their compatibility and exact rendered behavior require a validation phase.
- `redis_secure_password_123` and `fastapi_password` are assumed compromised due to source visibility and must be rotated rather than reused.
- The intended service account, PostgreSQL version, Redis/Memcached bind addresses, and desired Nginx-to-FastAPI routing are unspecified.
- The Vagrant box/network discrepancy and the printed `192.168.56.10` endpoint must be resolved before acceptance testing.
- A 2–3 week initial migration estimate assumes one infrastructure engineer, one application owner available for decisions, and access to a test VM; production hardening may extend the schedule by 1–2 weeks.
- Team coordination should use role ownership (web/security, cache, application/database), a shared variable contract, review of all Vault values, CI checks (`ansible-lint`, YAML/template validation, Molecule or Vagrant convergence), and an explicit sign-off checklist for SSH access, firewall behavior, TLS, database connectivity, and application health.
