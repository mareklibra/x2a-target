Migration Summary for fastapi_tutorial:
  Total items: 11
  Completed: 11
  Pending: 0
  Missing: 0
  Errors: 0
  Write attempts: 1
  Validation attempts: 0

Final Validation Report:
All migration tasks have been completed successfully

Validation passed with warnings:
ansible-lint: Passed with 2 warning(s):
[HIGH] tasks/main.yml:22 [no-changed-when] Commands should not change things if nothing needs doing. (Task/Handler: Install Python dependencies)
[HIGH] tasks/main.yml:31 [no-changed-when] Commands should not change things if nothing needs doing. (Task/Handler: Create database user and database)

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
## Checklist: fastapi_tutorial

### Templates
- [x] N/A → ./ansible/roles/fastapi_tutorial/templates/env.j2 (complete) - Created environment configuration template
- [x] N/A → ./ansible/roles/fastapi_tutorial/templates/fastapi-tutorial.service.j2 (complete) - Created systemd service template

### Recipes → Tasks
- [x] cookbooks/fastapi-tutorial/recipes/default.rb → ./ansible/roles/fastapi_tutorial/tasks/main.yml (complete) - Converted Chef recipe to Ansible tasks

### Structure Files
- [x] N/A → ./ansible/roles/fastapi_tutorial/meta/main.yml (complete) - Created meta/main.yml file
- [x] N/A → ./ansible/roles/fastapi_tutorial/handlers/main.yml (complete) - Created handlers/main.yml file
- [x] N/A → ./ansible/roles/fastapi_tutorial/defaults/main.yml (complete) - Created defaults/main.yml file
- [x] cookbooks/fastapi-tutorial/metadata.rb → ./ansible/roles/fastapi_tutorial/meta/main.yml (complete) - Converted metadata.rb to meta/main.yml
- [x] N/A → ansible/roles/fastapi_tutorial/meta/main.yml (complete)

### Dependencies (requirements.yml)
- [x] collection:ansible.posix → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Added ansible.posix collection to requirements.yml
- [x] collection:ansible.builtin → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Added ansible.builtin collection to requirements.yml
- [x] collection:community.postgresql → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Added community.postgresql collection to requirements.yml


Telemetry:
Phase: migrate
Duration: 0.00s

Agent Metrics:
  AAPDiscoveryAgent: 13.12s
    Tokens: 19688 in, 483 out
    Tools: aap_list_collections: 1, aap_search_collections: 2
    collections_found: 0
  PlanningAgent: 46.48s
    Tokens: 89590 in, 2321 out
    Tools: add_checklist_task: 12, list_checklist_tasks: 2, list_directory: 2
  WriteAgent: 117.35s
    Tokens: 377554 in, 6491 out
    Tools: ansible_lint: 4, ansible_write: 7, get_checklist_summary: 1, list_checklist_tasks: 2, read_file: 2, update_checklist_task: 10, write_file: 2
    attempts: 1
    complete: True
    files_created: 11
    files_total: 11
  ValidationAgent: 11.78s
    collections_installed: 2
    collections_failed: 0
    validators_passed: ['ansible-lint', 'role-check']
    validators_failed: []
    attempts: 0
    complete: True
    has_errors: False