# MIGRATION FROM CHEF TO ANSIBLE

The repository is a small Chef Solo infrastructure repository with three local cookbooks, external Supermarket cookbook dependencies, and a Vagrant/libvirt test environment. The migration is modest in module count but has meaningful security and platform risks: credentials are embedded in recipes, TLS keys are generated locally as self-signed certificates, and the Vagrant box (Fedora 42) conflicts with the Debian/Ubuntu provisioning commands and several Ubuntu-specific paths. A two- to four-week effort is a reasonable estimate for one engineer, including implementation, platform correction, security remediation, and validation; parallel work by an Ansible engineer and an application/security reviewer could shorten this to approximately two weeks.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server configuration for three SSL-enabled virtual hosts (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), with static content, HTTP-to-HTTPS redirects, security headers, rate limiting, logging, and host hardening.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef
    - Key Features: Nginx installation and service management, templated `nginx.conf` and per-site configurations, self-signed certificate generation, Fail2ban, UFW, SSH hardening, sysctl hardening, static site files, and custom line-editing resource.
    - Ansible target: A role or role set for nginx, TLS, security baseline, firewall, and virtual hosts; use handlers for nginx/fail2ban/SSH reloads and `ansible.builtin.template`, `copy`, `package`, `service`, `file`, `ufw`, `sysctl`, and certificate modules as appropriate.

- **cache**:
    - Description: Installs and enables Memcached and Redis, creates a Redis log directory, configures a Redis server on port 6379, and applies a post-generation configuration workaround.
    - Path: `cookbooks/cache`
    - Technology: Chef
    - Key Features: External `memcached` and `redisio` cookbook recipes, Redis password authentication, Redis service enablement, log directory ownership, and a Ruby-based cleanup of replication-related settings in `/etc/redis/6379.conf`.
    - Ansible target: A cache role using distribution-compatible Redis and Memcached packages/services, a managed Redis configuration template, `systemd` handlers, and an explicit decision about whether the current replication workaround remains necessary.

- **fastapi-tutorial**:
    - Description: Installs a Python FastAPI tutorial application from GitHub, creates a virtual environment, installs requirements, provisions PostgreSQL and a database user/database, writes an application `.env`, and runs Uvicorn as a systemd service on port 8000.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef
    - Key Features: Python 3 and pip/venv setup, Git checkout of `https://github.com/dibanez/fastapi_tutorial.git` at `main`, PostgreSQL service management, database bootstrap, systemd unit creation, and application startup configuration.
    - Ansible target: An application role separated from a PostgreSQL role, with idempotent `community.postgresql` database/user tasks, a least-privilege service account, a templated systemd unit, pinned application dependencies, and Ansible Vault-backed environment configuration.

**CRITICAL PATH VERIFICATION:**
All listed cookbook paths and entrypoints were present in the supplied repository tree and were read before inclusion. No external cookbook is listed as a local module.

### Infrastructure Files

- `Berksfile`: Chef dependency source and lockless dependency declarations. It includes local cookbooks plus `nginx ~> 12.0`, `memcached ~> 6.0`, and `redisio ~> 7.2.4`. Replace dependency resolution with Ansible Galaxy requirements and roles, or maintained internal roles; verify feature parity rather than assuming cookbook behavior maps one-to-one.
- `solo.rb`: Chef Solo configuration, including cookbook paths, cache path, logging, and stdout output. Replace with Ansible inventory, `ansible.cfg`, group variables, and role paths.
- `solo.json`: Chef run list and all active site/security variables. Convert to inventory/group/host vars and role defaults/vars. The JSON values override cookbook defaults for document roots and define the three virtual hosts.
- `Vagrantfile`: Vagrant test harness using `generic/fedora42`, hostname `chef-nginx`, libvirt, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, ports 80/443 forwarded to 8080/8443, and an rsync mount at `/chef-repo`. Preserve or replace with an Ansible provisioner and an explicit supported OS matrix.
- `vagrant-provision.sh`: Installs packages with `apt-get`, installs Chef from the Omnitruck script, installs Berkshelf, vendors dependencies, and runs Chef Solo. It must be replaced by `ansible-playbook` invocation or a controlled CI/test entrypoint. Its Debian assumptions do not match the Fedora box.
- `cookbooks/nginx-multisite/attributes/default.rb`: Defines default site roots, certificate paths, and security toggles. Convert to role defaults, with environment-specific values in inventory/group vars.
- `cookbooks/nginx-multisite/templates/`: Nginx, Fail2ban, sysctl, and virtual-host configuration templates. Port these to Jinja2 templates after validating syntax and desired security policy.
- `cookbooks/nginx-multisite/files/default/`: Static `index.html` files for the three sites. Copy into the corresponding document roots or make application content a separately managed concern.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: Custom Chef resource not referenced by the reviewed recipes. Determine whether it is dead code; if required, replace it with Ansible `lineinfile`/`blockinfile` or preferably a complete managed template.

### Target Details

- **Operating System**: The declared cookbook support is Ubuntu >=18.04 and CentOS >=7. The Vagrant box is explicitly Fedora 42, while `vagrant-provision.sh` uses `apt-get`, `/var/log/auth.log`, UFW, `www-data`, and Debian-style Nginx/PostgreSQL paths. The target OS is therefore unresolved. Establish a supported target before implementation; if no decision exists, use the migration default of Red Hat Enterprise Linux 9 and adapt firewall, package, service, user/group, and log-path behavior. Alternatively, explicitly change the lab box to Ubuntu if Debian compatibility is the actual requirement.
- **Virtual Machine Technology**: Vagrant with the libvirt provider; 2 vCPUs and 2 GB RAM. The private network is `192.168.121.10`, with HTTP/HTTPS forwarded to host ports 8080/8443.
- **Cloud Platform**: Not specified. No AWS, Azure, or GCP integration was found.

## Migration Approach

Create an Ansible project with `inventory/`, `group_vars/`, `host_vars/`, `roles/`, and `site.yml`. Keep application, cache, web, and security responsibilities separable even if they are initially executed by one play. Use check mode, Molecule or equivalent disposable-VM tests, `nginx -t`, systemd status checks, PostgreSQL connectivity tests, Redis authentication tests, and HTTP/HTTPS smoke tests. Preserve the existing three-host functional behavior, but do not preserve unsafe secrets or root-running application behavior merely for compatibility.

### Key Dependencies to Address

- **Chef Solo, Berkshelf, and Chef >=16**: Remove the Chef runtime and replace cookbook execution with Ansible playbooks and roles. Record all required behavior from the external cookbooks before retiring them.
- **`nginx` cookbook `~> 12.0`**: Replace with an internal Ansible nginx role and managed templates. Validate package names, module includes, virtual-host layout, reload behavior, and TLS defaults on the selected OS.
- **`memcached` cookbook `~> 6.0`**: Replace with distribution packages or a maintained Ansible role; define bind address, memory, and firewall exposure explicitly.
- **`redisio` cookbook `~> 7.2.4`**: Replace with a maintained Redis role or native tasks/template. Preserve port and authentication only after moving the password to Vault. Reassess the recipe's direct file mutation and replication setting removal.
- **PostgreSQL packages and client tooling**: Use distribution packages or a supported PostgreSQL role and `community.postgresql`; avoid shell commands with `|| true` because they conceal failed provisioning and are not safely idempotent.
- **Python/pip dependencies from the GitHub repository**: Pin the Git revision and dependency versions, use a controlled package/index policy where required, and validate whether the upstream repository is production appropriate.
- **Vagrant/libvirt**: Retain for local verification only or replace with an Ansible-compatible image/test pipeline. Resolve the Fedora-versus-Debian mismatch first.

### Security Considerations

- **Hardcoded Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Move it to Ansible Vault or an external secret manager, restrict Redis bind/firewall exposure, and rotate it during migration.
- **Hardcoded PostgreSQL credential**: `fastapi-tutorial` embeds `fastapi_password` in SQL and the `.env` connection string. Store database credentials in Vault, create the user with an idempotent PostgreSQL module, restrict `.env` to the service account with mode 0600, and rotate the credential.
- **Root-running application**: The systemd unit runs FastAPI as `root`. Create a dedicated non-privileged service user, set ownership of the application directory, and limit filesystem/network permissions.
- **Self-signed certificates**: The SSL recipe generates one-year RSA certificates with a fixed example subject. Treat these as development-only. For production, integrate the organization CA, ACME/Let's Encrypt, or certificate deployment from a secret store; protect private keys and enforce renewal/rollback procedures.
- **TLS and HTTP security**: Existing TLS 1.2/1.3 settings, HSTS, security headers, and certificate permissions need validation against current organizational standards. Review the cipher list, CSP use of `unsafe-inline`, certificate/key ownership, and whether HSTS is safe before production rollout.
- **Firewall and SSH hardening**: The source denies inbound traffic by default, allows SSH/HTTP/HTTPS, disables root SSH login, and disables password authentication. Reimplement using the selected OS's supported firewall module (UFW on Debian or firewalld on RHEL/Fedora), ensure console/break-glass access, and test the SSH handler before enforcing it.
- **Fail2ban and sysctl hardening**: Preserve the configured jails and kernel settings only after confirming log paths, IPv6 policy, and application/network requirements on the target OS. Disabling IPv6 globally may break future network designs.
- **File permissions and content**: Review public static content, Nginx logs, backup behavior, and the custom file-edit resource. Avoid shell interpolation for site names or paths and validate all variables before templating.
- **Secret scanning and rotation**: Run secret scanning in CI, remove credentials from Git history if this repository is shared, and document ownership and rotation for Redis, PostgreSQL, and TLS keys.

### Technical Challenges

- **Operating-system inconsistency**: Fedora 42 cannot execute the current `apt-get` bootstrap as written, and the recipes assume Debian users/paths and UFW. Mitigation: choose RHEL-family or Debian-family as an explicit target, then use `ansible_facts.os_family` only where multi-platform support is genuinely required.
- **External cookbook behavior is not fully visible**: The repository declares external cookbooks but does not contain their implementations. Mitigation: capture the exact resolved Berkshelf dependency set and test current behavior before replacing it.
- **Chef resource idempotence versus shell commands**: Redis config mutation, UFW commands, SSH `sed`, and PostgreSQL `psql || true` can conceal state and failures. Mitigation: use native Ansible modules and validation commands, with handlers and explicit changed conditions.
- **Conflicting document roots**: Cookbook defaults use `/opt/server/{test,ci,status}`, while `solo.json` overrides them to `/var/www/{...}`. Mitigation: define one authoritative inventory model and test that static files, virtual hosts, and permissions use the same roots.
- **Application deployment lifecycle**: Cloning `main` and installing unpinned requirements is not reproducible. Mitigation: pin a commit/release, build or verify a dependency lock, and define update/rollback procedures.
- **TLS generation and renewal**: Existing certificates are generated conditionally and never renewed. Mitigation: choose a certificate authority strategy and test renewal plus Nginx reload before production.
- **Port and DNS validation**: The Vagrant script reports an obsolete-looking `192.168.56.10` URL while the Vagrantfile uses `192.168.121.10`, and local host entries are commented out. Mitigation: correct documentation and add deterministic DNS/hosts setup to acceptance tests.

### Migration Order

1. **Platform and test foundation**: Decide the supported OS, correct the Vagrant image/provisioning path, build Ansible inventory, and establish linting, secret scanning, check-mode, and disposable-VM acceptance tests.
2. **Security baseline**: Implement package facts, SSH policy, firewall, sysctl, Fail2ban, service accounts, and secret storage. Validate out-of-band access before enforcing SSH changes.
3. **nginx-multisite**: Port templates, static files, three virtual hosts, handlers, TLS integration, and security headers. Test HTTP redirects, HTTPS, certificate permissions, and Nginx configuration validity.
4. **cache**: Deploy Redis and Memcached with secure authentication, explicit binding/firewall policy, and configuration validation. Remove the direct file workaround if a supported configuration template makes it unnecessary.
5. **fastapi-tutorial**: Deploy PostgreSQL and the application with pinned source/dependencies, Vault-managed credentials, a dedicated user, idempotent database creation, and systemd readiness/restart behavior.
6. **Cutover and decommissioning**: Compare Chef and Ansible convergence on a clean VM, perform service/data backup and rollback rehearsals, migrate secrets/certificates, then retire Chef/Berkshelf after an agreed observation period.

### Assumptions

- The three local cookbook directories are the complete module inventory; external Supermarket cookbooks are dependencies, not local modules.
- `solo.json` is the intended active configuration and therefore its `/var/www` roots override the cookbook defaults.
- The FastAPI GitHub repository and its `app.main:app` entrypoint remain available, although this was not inspected in the source repository.
- The target deployment is currently a single Vagrant-style host; production topology, HA, DNS, monitoring, backups, and load balancing are not specified.
- Fedora 42 is either a test-box mistake or an intentional target; the owner must resolve this before role implementation. RHEL 9 is the default fallback, not a fact established by the repository.
- Redis and PostgreSQL passwords shown in source are compromised or non-production examples and must be rotated, regardless of apparent use.
- Self-signed certificates are intended for development only; production certificate ownership and renewal are unspecified.
- No data migration requirement, existing database contents, external secret manager, CI platform, or compliance baseline was provided.
- The declared external cookbook versions are constraints, not proof of current compatibility or exact transitive dependency versions; lock and archive the resolved set before behavior comparison.
- The custom `lineinfile` resource appears unused by the reviewed recipes and can be removed unless omitted files invoke it.
- Firewall rules allowing SSH, HTTP, and HTTPS are intended, while Redis, Memcached, PostgreSQL, and FastAPI port 8000 should remain local unless an explicit architecture requires otherwise.
- Team coordination should assign owners for platform/security, web/TLS, cache, and application/database roles; require peer review for secrets and SSH/firewall changes; and maintain a behavior matrix mapping each Chef resource to an Ansible task, handler, test, and rollback procedure.
