Migration Summary for nginx_multisite:
  Total items: 19
  Completed: 19
  Pending: 0
  Missing: 0
  Errors: 0
  Write attempts: 1
  Validation attempts: 0

Final Validation Report:
All migration tasks have been completed successfully

All validations passed

Final checklist:
## Checklist: nginx_multisite

### Templates
- [x] cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb → ./ansible/roles/nginx_multisite/templates/fail2ban.jail.local.j2 (complete) - Converted fail2ban.jail.local template. No ERB variables were present, so content was copied as-is.
- [x] cookbooks/nginx-multisite/templates/default/nginx.conf.erb → ./ansible/roles/nginx_multisite/templates/nginx.conf.j2 (complete) - Converted nginx.conf template. No ERB variables were present, so content was copied as-is.
- [x] cookbooks/nginx-multisite/templates/default/security.conf.erb → ./ansible/roles/nginx_multisite/templates/security.conf.j2 (complete) - Converted security.conf template. No ERB variables were present, so content was copied as-is.
- [x] cookbooks/nginx-multisite/templates/default/site.conf.erb → ./ansible/roles/nginx_multisite/templates/site.conf.j2 (complete) - Converted site.conf template. ERB variables were converted to Jinja2 format: @server_name → server_name, @document_root → document_root, @cert_file → cert_file, @key_file → key_file, and if/else statements were converted to Jinja2 syntax.
- [x] cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb → ./ansible/roles/nginx_multisite/templates/sysctl-security.conf.j2 (complete) - Converted sysctl-security.conf template. No ERB variables were present, so content was copied as-is.

### Recipes → Tasks
- [x] cookbooks/nginx-multisite/recipes/default.rb → ./ansible/roles/nginx_multisite/tasks/main.yml (complete) - Converted default.rb to main.yml. Included all required task files in the correct order.
- [x] cookbooks/nginx-multisite/recipes/nginx.rb → ./ansible/roles/nginx_multisite/tasks/nginx.yml (complete) - Converted nginx.rb to nginx.yml. Implemented package installation, template configuration, service management, and document root creation with index files.
- [x] cookbooks/nginx-multisite/recipes/security.rb → ./ansible/roles/nginx_multisite/tasks/security.yml (complete) - Converted security.rb to security.yml. Implemented security package installation, fail2ban configuration, UFW firewall setup, sysctl security settings, and SSH hardening.
- [x] cookbooks/nginx-multisite/recipes/sites.rb → ./ansible/roles/nginx_multisite/tasks/sites.yml (complete) - Converted sites.rb to sites.yml. Implemented virtual host configuration, symlink creation, and default site removal.
- [x] cookbooks/nginx-multisite/recipes/ssl.rb → ./ansible/roles/nginx_multisite/tasks/ssl.yml (complete) - Converted ssl.rb to ssl.yml. Implemented SSL package installation, directory creation, and self-signed certificate generation.

### Attributes → Variables
- [x] cookbooks/nginx-multisite/attributes/default.rb → ./ansible/roles/nginx_multisite/defaults/main.yml (complete) - Converted attributes/default.rb to defaults/main.yml. Converted Ruby hash syntax to YAML format.

### Structure Files
- [x] N/A → ./ansible/roles/nginx_multisite/meta/main.yml (complete) - Created meta/main.yml with role metadata including description, license, platforms, and tags.
- [x] N/A → ./ansible/roles/nginx_multisite/handlers/main.yml (complete) - Created handlers/main.yml with handlers for nginx reload/restart, fail2ban restart, ssh restart, and sysctl reload.
- [x] N/A → ./ansible/roles/nginx_multisite/tasks/main.yml (complete) - This task is redundant as tasks/main.yml was already created and marked as complete in the Recipes → Tasks section.
- [x] N/A → ./ansible/roles/nginx_multisite/defaults/main.yml (complete) - This task is redundant as defaults/main.yml was already created and marked as complete in the Attributes → Variables section.
- [x] N/A → ansible/roles/nginx_multisite/meta/main.yml (complete)

### Dependencies (requirements.yml)
- [x] collection:ansible.posix → ./ansible/roles/nginx_multisite/requirements.yml (complete) - Created requirements.yml with required collections: ansible.posix, community.general, and community.crypto.
- [x] collection:community.general → ./ansible/roles/nginx_multisite/requirements.yml (complete) - Added community.general collection to requirements.yml.
- [x] collection:community.crypto → ./ansible/roles/nginx_multisite/requirements.yml (complete) - Added community.crypto collection to requirements.yml.


Telemetry:
Phase: migrate
Duration: 0.00s

Agent Metrics:
  AAPDiscoveryAgent: 12.42s
    Tokens: 24051 in, 488 out
    Tools: aap_list_collections: 1, aap_search_collections: 2
    collections_found: 0
  PlanningAgent: 56.76s
    Tokens: 151923 in, 3129 out
    Tools: add_checklist_task: 18, list_checklist_tasks: 2
  WriteAgent: 200.10s
    Tokens: 282825 in, 1932 out
    Tools: ansible_lint: 1, ansible_write: 2, get_checklist_summary: 1, list_checklist_tasks: 2, update_checklist_task: 7
    attempts: 1
    complete: True
    files_created: 19
    files_total: 19
  ValidationAgent: 14.06s
    collections_installed: 3
    collections_failed: 0
    validators_passed: ['ansible-lint', 'role-check']
    validators_failed: []
    attempts: 0
    complete: True
    has_errors: False