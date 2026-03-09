Migration Summary for nginx_multisite:
  Total items: 20
  Completed: 20
  Pending: 0
  Missing: 0
  Errors: 0
  Write attempts: 1
  Validation attempts: 0

Final Validation Report:
All migration tasks have been completed successfully

Validation passed with warnings:
ansible-lint: Passed with 1 warning(s):
[HIGH] handlers/main.yml:17 [no-changed-when] Commands should not change things if nothing needs doing. (Task/Handler: Reload sysctl)

==============================
Rule Hints (How to Fix):
==============================
# no-changed-when

Commands should use `changed_when` to indicate when they actually change something.

## Problematic code

```yaml
- name: Does not handle any output or return codes
  ansible.builtin.command: cat {{ my_file | quote }}
```

## Correct code

```yaml
- name: Handle command output
  ansible.builtin.command: cat {{ my_file | quote }}
  register: my_output
  changed_when: my_output.rc != 0
```

Common patterns:
- `changed_when: false` - Task never changes anything
- `changed_when: true` - Task always changes something
- `changed_when: result.rc != 0` - Use command result to determine change

Final checklist:
## Checklist: nginx_multisite

### Templates
- [x] cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb → ./ansible/roles/nginx_multisite/templates/fail2ban.jail.local.j2 (complete) - Converted fail2ban jail configuration from ERB to Jinja2. No template variables were present in the original file.
- [x] cookbooks/nginx-multisite/templates/default/nginx.conf.erb → ./ansible/roles/nginx_multisite/templates/nginx.conf.j2 (complete) - Converted nginx.conf from ERB to Jinja2. No template variables were present in the original file.
- [x] cookbooks/nginx-multisite/templates/default/security.conf.erb → ./ansible/roles/nginx_multisite/templates/security.conf.j2 (complete) - Converted security.conf from ERB to Jinja2. No template variables were present in the original file.
- [x] cookbooks/nginx-multisite/templates/default/site.conf.erb → ./ansible/roles/nginx_multisite/templates/site.conf.j2 (complete) - Converted site.conf from ERB to Jinja2. Replaced ERB variables (@server_name, @document_root, @ssl_enabled, @cert_file, @key_file) with Jinja2 variables (removed @ prefix).
- [x] cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb → ./ansible/roles/nginx_multisite/templates/sysctl-security.conf.j2 (complete) - Converted sysctl-security.conf from ERB to Jinja2. No template variables were present in the original file.

### Recipes → Tasks
- [x] cookbooks/nginx-multisite/recipes/default.rb → ./ansible/roles/nginx_multisite/tasks/main.yml (complete) - Converted default.rb to main.yml using ansible.builtin.import_tasks to include other task files.
- [x] cookbooks/nginx-multisite/recipes/nginx.rb → ./ansible/roles/nginx_multisite/tasks/nginx.yml (complete) - Converted nginx.rb to nginx.yml. Implemented nginx package installation, configuration of nginx.conf and security.conf, service management, and document root setup for each site.
- [x] cookbooks/nginx-multisite/recipes/security.rb → ./ansible/roles/nginx_multisite/tasks/security.yml (complete) - Converted security.rb to security.yml. Implemented package installation, fail2ban configuration, UFW firewall setup, sysctl security parameters, and SSH hardening.
- [x] cookbooks/nginx-multisite/recipes/sites.rb → ./ansible/roles/nginx_multisite/tasks/sites.yml (complete) - Converted sites.rb to sites.yml. Implemented virtual host configuration creation, symlink creation for site enabling, and removal of default site.
- [x] cookbooks/nginx-multisite/recipes/ssl.rb → ./ansible/roles/nginx_multisite/tasks/ssl.yml (complete) - Converted ssl.rb to ssl.yml. Implemented SSL package installation, directory creation, and self-signed certificate generation for each site.

### Attributes → Variables
- [x] cookbooks/nginx-multisite/attributes/default.rb → ./ansible/roles/nginx_multisite/defaults/main.yml (complete) - Converted default.rb attributes to defaults/main.yml. Defined nginx sites configuration, SSL paths, and security settings.

### Static Files
- [x] cookbooks/nginx-multisite/files/default/test/index.html → ./ansible/roles/nginx_multisite/files/test/index.html (complete) - Copied test/index.html static file.
- [x] cookbooks/nginx-multisite/files/default/ci/index.html → ./ansible/roles/nginx_multisite/files/ci/index.html (complete) - Copied ci/index.html static file.
- [x] cookbooks/nginx-multisite/files/default/status/index.html → ./ansible/roles/nginx_multisite/files/status/index.html (complete) - Copied status/index.html static file.

### Structure Files
- [x] N/A → ./ansible/roles/nginx_multisite/meta/main.yml (complete) - Already completed in previous task.
- [x] N/A → ./ansible/roles/nginx_multisite/handlers/main.yml (complete) - Created handlers/main.yml with handlers for nginx, fail2ban, ssh, and sysctl.
- [x] N/A → ./ansible/roles/nginx_multisite/defaults/main.yml (complete) - Already completed in previous task.
- [x] N/A → ./ansible/roles/nginx_multisite/tasks/main.yml (complete) - Already completed in previous task.
- [x] cookbooks/nginx-multisite/metadata.rb → ./ansible/roles/nginx_multisite/meta/main.yml (complete) - Created meta/main.yml with role metadata based on the Chef metadata.rb file.
- [x] N/A → ansible/roles/nginx_multisite/meta/main.yml (complete)


Telemetry:
Phase: migrate
Duration: 0.00s

Agent Metrics:
  AAPDiscoveryAgent: 13.51s
    Tools: aap_list_collections: 1, aap_search_collections: 3
    collections_found: 0
  PlanningAgent: 80.43s
    Tools: add_checklist_task: 19, list_checklist_tasks: 2, list_directory: 11
  WriteAgent: 266.52s
    Tools: ansible_lint: 6, ansible_write: 7, copy_file: 3, get_checklist_summary: 1, list_checklist_tasks: 1, update_checklist_task: 8
    attempts: 1
    complete: True
    files_created: 20
    files_total: 20
  ValidationAgent: 1.73s
    validators_passed: ['ansible-lint', 'role-check']
    validators_failed: []
    attempts: 0
    complete: True
    has_errors: False