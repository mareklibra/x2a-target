Migration Summary for fastapi_tutorial:
  Total items: 10
  Completed: 10
  Pending: 0
  Missing: 0
  Errors: 0
  Write attempts: 1
  Validation attempts: 0

Final Validation Report:
All migration tasks have been completed successfully

All validations passed

Final checklist:
## Checklist: fastapi_tutorial

### Recipes → Tasks
- [x] cookbooks/fastapi-tutorial/recipes/default.rb → ./ansible/roles/fastapi_tutorial/tasks/main.yml (complete) - Created tasks/main.yml with all required tasks from the Chef recipe

### Static Files
- [x] N/A → ./ansible/roles/fastapi_tutorial/templates/env.j2 (complete) - Created env.j2 template for application environment configuration
- [x] N/A → ./ansible/roles/fastapi_tutorial/templates/fastapi-tutorial.service.j2 (complete) - Created fastapi-tutorial.service.j2 template for systemd service configuration

### Structure Files
- [x] cookbooks/fastapi-tutorial/metadata.rb → ./ansible/roles/fastapi_tutorial/meta/main.yml (complete) - Created meta/main.yml with role metadata
- [x] N/A → ./ansible/roles/fastapi_tutorial/handlers/main.yml (complete) - Created handlers/main.yml with systemd reload and service restart handlers
- [x] N/A → ./ansible/roles/fastapi_tutorial/defaults/main.yml (complete) - Created defaults/main.yml with all configurable variables
- [x] N/A → ansible/roles/fastapi_tutorial/meta/main.yml (complete)

### Dependencies (requirements.yml)
- [x] collection:ansible.posix → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Added ansible.posix collection to requirements.yml
- [x] collection:community.postgresql → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Added community.postgresql collection to requirements.yml
- [x] collection:ansible.builtin → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - ansible.builtin is included by default in Ansible, no need to add to requirements.yml


Telemetry:
Phase: migrate
Duration: 0.00s

Agent Metrics:
  AAPDiscoveryAgent: 10.91s
    Tokens: 18944 in, 448 out
    Tools: aap_list_collections: 1, aap_search_collections: 2
    collections_found: 0
  PlanningAgent: 36.98s
    Tokens: 77654 in, 2031 out
    Tools: add_checklist_task: 10, list_checklist_tasks: 2, list_directory: 2
  WriteAgent: 103.16s
    Tokens: 362588 in, 6272 out
    Tools: ansible_doc_lookup: 3, ansible_lint: 2, ansible_write: 7, list_checklist_tasks: 2, read_file: 2, update_checklist_task: 9, write_file: 2
    attempts: 1
    complete: True
    files_created: 10
    files_total: 10
  ValidationAgent: 7.78s
    collections_installed: 2
    collections_failed: 0
    validators_passed: ['ansible-lint', 'role-check']
    validators_failed: []
    attempts: 0
    complete: True
    has_errors: False