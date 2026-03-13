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

Validation passed with warnings:
ansible-lint: Passed with 1 warning(s):
[MEDIUM] vars/main.yml:3 [yaml] No new line character at the end of file ()

==============================
Rule Hints (How to Fix):
==============================
# yaml

Checks YAML syntax for indentation and formatting issues.

## Common indentation issues

### Problematic code

```yaml
# Incorrect indentation
- name: Configure service
  service:
  name: nginx  # <- Should be indented under service
  state: started
```

```yaml
# Inconsistent indentation
- name: Install packages
  apt:
    name: nginx
      state: present  # <- Too much indentation
```

```yaml
# Comment indentation
- name: Task
  debug:
    msg: "test"
      # <- Comment indented incorrectly
```

### Correct code

```yaml
# Correct indentation
- name: Configure service
  service:
    name: nginx  # <- Properly indented
    state: started
```

```yaml
# Consistent indentation
- name: Install packages
  apt:
    name: nginx
    state: present  # <- Aligned with name
```

```yaml
# Comment indentation
- name: Task
  debug:
    msg: "test"
  # <- Comment at correct level
```

## Other common issues

### Octal values

```yaml
# Problematic
permissions: 0777  # <- yaml[octal-values]

# Correct
permissions: "0777"  # <- Quote octal values
```

### Duplicate keys

```yaml
# Problematic
foo: value1
foo: value2  # <- yaml[key-duplicates]

# Correct
foo: value2  # <- Use unique keys
```

Final checklist:
## Checklist: fastapi_tutorial

### Recipes → Tasks
- [x] cookbooks/fastapi-tutorial/recipes/default.rb → ./ansible/roles/fastapi_tutorial/tasks/main.yml (complete) - Converted Chef recipe to Ansible tasks

### Static Files
- [x] N/A → ./ansible/roles/fastapi_tutorial/templates/env.j2 (complete) - Created environment file template
- [x] N/A → ./ansible/roles/fastapi_tutorial/templates/fastapi-tutorial.service.j2 (complete) - Created systemd service template

### Structure Files
- [x] cookbooks/fastapi-tutorial/metadata.rb → ./ansible/roles/fastapi_tutorial/meta/main.yml (complete) - Created meta/main.yml from metadata.rb
- [x] N/A → ./ansible/roles/fastapi_tutorial/handlers/main.yml (complete) - Created handlers file with systemd reload handler
- [x] N/A → ./ansible/roles/fastapi_tutorial/defaults/main.yml (complete) - Created defaults file with configurable variables
- [x] N/A → ./ansible/roles/fastapi_tutorial/vars/main.yml (complete) - Created vars file for non-overridable variables
- [x] N/A → ansible/roles/fastapi_tutorial/meta/main.yml (complete)

### Dependencies (requirements.yml)
- [x] collection:community.postgresql → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Added community.postgresql collection requirement
- [x] collection:ansible.posix → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Added ansible.posix collection requirement


Telemetry:
Phase: migrate
Duration: 0.00s

Agent Metrics:
  AAPDiscoveryAgent: 11.82s
    Tokens: 18927 in, 444 out
    Tools: aap_list_collections: 1, aap_search_collections: 2
    collections_found: 0
  PlanningAgent: 36.26s
    Tokens: 71397 in, 1962 out
    Tools: add_checklist_task: 9, list_checklist_tasks: 2, list_directory: 2
  WriteAgent: 89.39s
    Tokens: 312675 in, 4485 out
    Tools: ansible_lint: 2, ansible_write: 7, list_checklist_tasks: 1, read_file: 2, update_checklist_task: 9, write_file: 5
    attempts: 1
    complete: True
    files_created: 10
    files_total: 10
  ValidationAgent: 8.73s
    collections_installed: 2
    collections_failed: 0
    validators_passed: ['ansible-lint', 'role-check']
    validators_failed: []
    attempts: 0
    complete: True
    has_errors: False