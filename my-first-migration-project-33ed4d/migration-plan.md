# MIGRATION FROM CHEF TO ANSIBLE

This repository is a small Chef Solo/Vagrant environment with three local cookbooks and three external Supermarket dependencies. The target migration should preserve the current development topology—an Nginx multi-site edge, security hardening, Redis/memcached services, and a FastAPI/PostgreSQL application—while replacing Chef resources, recipes, attributes, and Berkshelf resolution with Ansible roles, playbooks, templates, handlers, and managed secrets.

The scope is modest in module count but moderate in operational risk because the cookbooks change firewall and SSH policy, generate TLS keys, create database credentials, install software from an external Git repository, and run services as root. A realistic implementation is approximately 1–2 weeks for an experienced Ansible engineer, including role development, Fedora/Ubuntu compatibility decisions, security remediation, and validation in Vagrant. A production-ready rollout with CI, secret-management integration, and staged testing is more realistically 2–4 weeks depending on the target platform and certificate/database requirements.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

**CRITICAL PATH VERIFICATION:**
Only modules whose paths were present in the repository tree are listed below.

- **nginx-multisite**:
    - Description: Nginx web-server cookbook that installs and enables Nginx, applies security and sysctl settings, configures three virtual hosts, serves static site content, and provisions development TLS certificates.
    - Path: `cookbooks/nginx-multisite`
    - Technology: Chef cookbook
    - Key Features: Three Hiera-equivalent attribute-driven sites (`test.cluster.local`, `ci.cluster.local`, and `status.cluster.local`), Nginx configuration templates, site enablement symlinks, fail2ban, UFW, SSH hardening, and self-signed RSA certificates.
    - Migration shape: An Ansible role with separate task files for Nginx, security, TLS, and sites; templates and handlers for Nginx/fail2ban/sshd/sysctl; a loop over a structured `nginx_sites` variable.

- **cache**:
    - Description: Caching-services cookbook that includes the external memcached and redisio cookbooks, creates the Redis log directory, configures a Redis server on port 6379, and enables Redis.
    - Path: `cookbooks/cache`
    - Technology: Chef cookbook
    - Key Features: Memcached installation through the `memcached` cookbook, Redis installation/configuration through `redisio`, Redis authentication, and a post-configuration workaround that removes selected replication-related directives.
    - Migration shape: An Ansible role using distribution packages or an explicitly selected Redis repository, service modules, a rendered Redis configuration, and a documented replacement for the Chef `ruby_block` workaround.

- **fastapi-tutorial**:
    - Description: FastAPI application provisioning cookbook that installs Python, PostgreSQL, build/database packages, clones the tutorial application from GitHub, creates a virtual environment, installs Python requirements, creates a PostgreSQL database/user, writes `.env`, and manages a systemd service.
    - Path: `cookbooks/fastapi-tutorial`
    - Technology: Chef cookbook
    - Key Features: Python virtualenv, PostgreSQL service and database initialization, Git-based application deployment, Uvicorn on `0.0.0.0:8000`, systemd integration, and application environment configuration.
    - Migration shape: An Ansible role with package installation, `ansible.builtin.git`, `pip`/virtualenv handling, PostgreSQL modules or controlled SQL tasks, a systemd unit template, and vaulted application credentials.

### Infrastructure Files

- `Berksfile`: Defines the Chef Supermarket source, three local cookbooks, and external dependencies `nginx ~> 12.0`, `memcached ~> 6.0`, and `redisio ~> 7.2.4`. Replace with Ansible Galaxy requirements only if third-party roles are deliberately adopted; otherwise implement small, owned roles to reduce dependency drift.
- `Vagrantfile`: Defines a `generic/fedora42` VM, hostname `chef-nginx`, libvirt provider, 2 GB RAM, 2 CPUs, private IP `192.168.121.10`, and HTTP/HTTPS forwarded ports. Retain or revise this as the Ansible integration-test harness.
- `solo.rb`: Chef Solo configuration with cookbook paths, cache directory `/var/chef-solo`, and stdout logging. Replace with an Ansible inventory, `ansible.cfg`, and playbook/role layout.
- `solo.json`: Chef run list and node attributes for the three cookbooks, site roots, certificate paths, and security settings. Convert to group/host variables, preferably split into non-secret service configuration and vaulted secrets.
- `vagrant-provision.sh`: Installs Chef and Berkshelf, vendors dependencies, and runs Chef Solo. Replace with an Ansible provisioner or shell bootstrap that installs Python/Ansible prerequisites only; dependency installation should occur in the control environment or through a pinned CI process.
- `cookbooks/nginx-multisite/templates/`: Contains Nginx, site, fail2ban, and sysctl templates. These are direct candidates for Ansible role templates, subject to validation and variable cleanup.
- `cookbooks/nginx-multisite/files/`: Contains static `index.html` content for the `ci`, `status`, and `test` sites. Move into the role's `files/` tree or, preferably, deploy application-owned web content separately.
- `cookbooks/nginx-multisite/resources/lineinfile.rb`: A custom Chef resource is present in the tree but is not referenced by the reviewed default, nginx, security, sites, or SSL recipes. Determine whether it is dead code before migration; do not automatically port it.

### Target Details

- **Operating System**: The Vagrant box is explicitly Fedora 42, but `vagrant-provision.sh` uses `apt-get`, and the cookbooks declare support for Ubuntu `>= 18.04` and CentOS `>= 7.0`. The current implementation is therefore internally inconsistent. The migration must select a supported target. For the supplied Vagrant box, use Fedora/RHEL-family package and service behavior (`dnf`, `firewalld`, and typically `sshd`); alternatively change the box to Ubuntu if UFW and the existing bootstrap are intended to remain. Do not assume the current Chef code works unchanged on Fedora.
- **Virtual Machine Technology**: Vagrant with the libvirt provider; the VM has 2 CPUs and 2048 MB RAM. No production virtualization platform is specified.
- **Cloud Platform**: Not specified. No AWS, Azure, or GCP configuration was identified.

## Migration Approach

Create a repository-level Ansible structure such as `site.yml`, `inventories/dev/hosts.yml`, `group_vars/`, `roles/nginx_multisite`, `roles/cache`, and `roles/fastapi_tutorial`. Preserve the current high-level order—security prerequisites, cache services, application/database, then Nginx/sites—or explicitly model service dependencies with handlers and health checks. Use tags (`security`, `nginx`, `cache`, `database`, `application`, `tls`) to support incremental rollout.

First establish the target OS and test the base connection. Then port configuration into variables, implement idempotent roles, introduce Vault or an external secret manager, and validate with Molecule or Vagrant. Acceptance tests should check enabled services, listening ports, Nginx syntax, all virtual hosts, certificate/key permissions, Redis authentication, PostgreSQL connectivity, FastAPI health, and SSH/firewall policy.

### Key Dependencies to Address

- **Chef Solo/Berkshelf**: Replace Chef Solo execution and Berkshelf vendoring with `ansible-playbook`, pinned Ansible collections, and CI-controlled dependency installation. Avoid an unpinned runtime download path.
- **nginx cookbook `~> 12.0`**: Replace with an owned `nginx_multisite` role using native package/service/template modules. Decide whether Fedora's Nginx package layout matches the existing `/etc/nginx/sites-available` and `sites-enabled` assumptions.
- **memcached cookbook `~> 6.0`**: Use the target OS package and service modules, with explicit listen address, memory, and firewall policy. Do not expose memcached broadly by default.
- **redisio cookbook `~> 7.2.4`**: Use a pinned OS package/repository or a reviewed Ansible collection/role. Render Redis configuration rather than relying on Chef cookbook defaults, and reproduce only the required configuration after understanding the replication-directive workaround.
- **PostgreSQL packages and SQL**: Use `community.postgresql` modules where possible, with the required Python PostgreSQL driver installed on the managed host. Make database/user creation idempotent and avoid shell commands with `|| true`.
- **GitHub repository `https://github.com/dibanez/fastapi_tutorial.git`, revision `main`**: Pin a release, tag, or commit for repeatable deployments; assess availability, ownership, and supply-chain risk before production use.
- **Python requirements from the cloned repository**: Install into a dedicated virtual environment with a locked requirements file or hashes, rather than allowing an unreviewed moving dependency set.
- **systemd and Uvicorn**: Convert the inline unit into a template and use `systemd_service`/`service` plus handlers. Decide whether the service should run as a dedicated unprivileged user instead of root.
- **OpenSSL/self-signed certificates**: For development, use an idempotent certificate task or `community.crypto`; for production, use organization-issued certificates or ACME automation. Do not treat generated development keys as production credentials.

### Security Considerations

- **Hardcoded Redis credential**: `cookbooks/cache/recipes/default.rb` contains `redis_secure_password_123`. Move the Redis password to Ansible Vault or an external secret manager, set restrictive file permissions, and rotate it during migration.
- **Hardcoded PostgreSQL credentials**: The FastAPI recipe embeds user `fastapi`, password `fastapi_password`, and the same password in a world-readable `.env` file (`0644`). Vault the password, write the environment file with restrictive permissions, and preferably use a systemd credential/environment mechanism or a root-readable file owned by the service user.
- **TLS private keys**: The SSL recipe creates self-signed keys under `/etc/ssl/private`, assigns `root:ssl-cert`, and uses mode `0640` after generation. Preserve least privilege, ensure only the Nginx process can read the key, and replace self-signed certificates with managed certificates where applicable. Certificate renewal and SAN coverage are not addressed by the source.
- **SSH hardening**: The security recipe disables root login and password authentication using `sed`. Reimplement with `ansible.builtin.lineinfile` or a drop-in, validate `sshd -t`, and ensure key-based access and a recovery path exist before applying the change.
- **Firewall policy**: The source enables UFW, denies by default, and allows SSH/HTTP/HTTPS. On Fedora 42, use firewalld rather than UFW unless UFW is intentionally installed and supported. Explicitly decide whether PostgreSQL, Redis, memcached, and FastAPI port 8000 must remain localhost-only; they should not be exposed merely because the application binds to all interfaces.
- **System hardening**: The sysctl template and fail2ban configuration should be reviewed for compatibility, expected kernel values, correct log paths, and safe defaults before deployment. Apply changes with validation and controlled handlers.
- **Application privilege**: The systemd unit runs Uvicorn as `root` and clones the application into a root-owned path. Create a dedicated application account and use narrowly scoped ownership and permissions.
- **Supply-chain and transport risk**: Chef installation uses `curl | bash`, Berkshelf downloads from Supermarket, and the application is cloned from the moving `main` branch. Replace these with pinned, verified artifacts and trusted package sources.
- **Secrets management**: No encrypted data bags or Chef Vault usage were identified. Visible credentials are embedded directly in Chef recipe code and `.env` content. The migration should introduce Vault from the first implementation, with separate development and production secret paths and a documented rotation process.

### Technical Challenges

- **OS mismatch**: Fedora 42 is declared by Vagrant while the bootstrap uses `apt-get` and the security implementation uses UFW. Resolve the platform decision before role coding; use OS-family conditionals only where genuinely required.
- **Nginx layout and configuration compatibility**: The source expects Debian-style `sites-available`/`sites-enabled` paths and `www-data`, which may differ on Fedora. Validate package paths, service names, user/group names, and template syntax on the chosen OS.
- **Chef ordering and hidden cookbook behavior**: External `nginx`, `memcached`, and `redisio` cookbooks may have provided implicit defaults. Inventory the rendered current state before cutover and make all required settings explicit in Ansible.
- **Redis workaround**: The Chef `ruby_block` deletes replication directives after the external cookbook runs. This is brittle and may indicate version-specific incompatibility. Identify the actual Redis version/configuration and replace it with a supported configuration model.
- **Database idempotency and migrations**: The source uses shell SQL commands with `|| true`, which can conceal failures and does not manage schema migrations. Separate role provisioning from application migrations and fail visibly on unexpected database errors.
- **Certificate lifecycle**: Certificates are generated only when files do not exist and have a one-year lifetime. Define renewal, replacement, hostname/SAN, backup, and trust requirements before production migration.
- **Application availability**: The recipe installs dependencies and starts FastAPI but does not define readiness checks, reverse proxy routing to port 8000, deployment rollback, or application upgrade strategy. Add health checks and decide whether Nginx should proxy any of the configured sites to FastAPI.
- **Unclear site-content mapping**: `solo.json` uses `/var/www/...` roots, while cookbook attributes default to `/opt/server/...`; the Nginx recipe consumes node attributes and the static files use cookbook-relative paths. Confirm the intended document roots and test each hostname.

### Migration Order

1. **Platform and baseline**: Decide Fedora versus Ubuntu, create the Ansible inventory and Vagrant execution path, and establish connectivity and package/service facts.
2. **Security baseline**: Port SSH policy, firewall, sysctl, fail2ban, TLS directory permissions, and validation checks. Test recovery access before enforcing SSH changes.
3. **cache**: Migrate memcached and Redis with vaulted credentials and localhost/network restrictions. Validate service health and authentication.
4. **fastapi-tutorial database/application**: Install PostgreSQL and Python prerequisites, create the dedicated database/user, deploy a pinned application revision, use a non-root service account, and add health checks.
5. **nginx-multisite**: Port Nginx, static content, virtual hosts, certificate handling, and reload handlers after the underlying services and platform conventions are stable.
6. **Integration and cutover**: Run idempotence tests twice, verify all three hostnames and forwarded ports, compare rendered configuration with the Chef environment, then retire Chef/Berkshelf provisioning.

### Assumptions

- The three directories containing `metadata.rb` and `recipes/default.rb` are the complete local cookbook inventory.
- `solo.json` is the effective Chef node configuration for the intended run list; no additional roles, environments, Chef Server policy, or data bags were provided.
- The external cookbook implementations were not included, so their exact rendered defaults, service names, and package behavior must be verified during discovery.
- The target is currently a development/test Vagrant VM rather than a declared production environment.
- Fedora 42 is intentional unless the team elects to change the Vagrant box to an Ubuntu image to match the existing apt/UFW assumptions.
- The three certificates are development self-signed certificates; production certificate ownership and trust requirements are unspecified.
- The application GitHub repository and its `requirements.txt` are external inputs and were not audited; their runtime, schema, and health endpoint must be confirmed.
- The intended relationship between the FastAPI service on port 8000 and the three Nginx sites is unclear; the source config only demonstrates static site roots.
- The `solo.json` document roots differ from cookbook attribute defaults, so the desired paths must be confirmed before migration.
- No cloud provider, load balancer, DNS automation, centralized logging, monitoring, backup policy, or disaster-recovery requirements are specified.
- No Ansible version, collection policy, CI platform, secret manager, certificate authority, or production inventory is specified; these must be selected and documented by the coordinating team.
- The unused-looking custom resource `resources/lineinfile.rb` may be obsolete, but its consumers should be confirmed before deletion.
- Migration estimates assume one engineer familiar with Ansible and access to a test VM; production approvals, security review, and application-owner testing may extend the schedule.
