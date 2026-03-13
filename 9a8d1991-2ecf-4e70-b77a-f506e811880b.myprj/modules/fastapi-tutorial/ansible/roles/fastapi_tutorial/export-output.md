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
[HIGH] tasks/install.yml:40 [args] Unsupported parameters for (basic.py) module: become, become_user. Supported parameters include: autocommit, ca_cert, connect_params, encoding, login_db, login_host, login_password, login_port, login_unix_socket, login_user, named_args, positional_args, query, search_path, session_role, ssl_cert, ssl_key, ssl_mode, trust_input (db, host, login, port, ssl_rootcert, unix_socket). (Task/Handler: Create database and user)
[MEDIUM] vars/main.yml:2 [yaml] No new line character at the end of file ()

==============================
Rule Hints (How to Fix):
==============================
# args

Validates task arguments against module documentation.

## Problematic code

```yaml
- name: Clone content repository
  ansible.builtin.git:  # Missing required 'repo' argument
    dest: /home/www
    version: master

- name: Enable service httpd
  ansible.builtin.systemd:  # Missing 'name' required by 'enabled'
    enabled: true

- name: Do not use mutually exclusive arguments
  ansible.builtin.command:
    cmd: /bin/echo  # cmd and argv are mutually exclusive
    argv:
      - Hello
```

## Correct code

```yaml
- name: Clone content repository
  ansible.builtin.git:
    repo: https://github.com/ansible/ansible-examples
    dest: /home/www
    version: master

- name: Enable service httpd
  ansible.builtin.systemd:
    name: httpd
    enabled: true

- name: Use command with cmd only
  ansible.builtin.command:
    cmd: "/bin/echo Hello"
```

Tip: Use `# noqa: args[module]` to skip validation when using complex jinja expressions.

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

### Templates
- [x] N/A → ./ansible/roles/fastapi_tutorial/templates/env.j2 (complete) - Created environment file template with variables for project name, API version, and database connection
- [x] N/A → ./ansible/roles/fastapi_tutorial/templates/fastapi-tutorial.service.j2 (complete) - Created systemd service template with variables for service user, app path, host, and port

### Recipes → Tasks
- [x] cookbooks/fastapi-tutorial/recipes/default.rb → ./ansible/roles/fastapi_tutorial/tasks/install.yml (complete) - Created install tasks file with package installation, git clone, Python venv setup, PostgreSQL configuration, and service setup

### Structure Files
- [x] cookbooks/fastapi-tutorial/metadata.rb → ./ansible/roles/fastapi_tutorial/meta/main.yml (complete) - Created meta/main.yml with role metadata from Chef cookbook
- [x] N/A → ./ansible/roles/fastapi_tutorial/tasks/main.yml (complete) - Created main tasks file that imports the install tasks
- [x] N/A → ./ansible/roles/fastapi_tutorial/defaults/main.yml (complete) - Created defaults/main.yml with configurable variables for FastAPI application, Git repository, and database settings
- [x] N/A → ./ansible/roles/fastapi_tutorial/handlers/main.yml (complete) - Created handlers/main.yml with systemd reload handler
- [x] N/A → ./ansible/roles/fastapi_tutorial/vars/main.yml (complete) - Created vars/main.yml with placeholder for non-overridable variables
- [x] N/A → ansible/roles/fastapi_tutorial/meta/main.yml (complete)

### Dependencies (requirements.yml)
- [x] collection:community.postgresql → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Created requirements.yml with community.postgresql and ansible.posix collections
- [x] collection:ansible.posix → ./ansible/roles/fastapi_tutorial/requirements.yml (complete) - Added ansible.posix collection to requirements.yml


Telemetry:
Phase: migrate
Duration: 0.00s

Agent Metrics:
  AAPDiscoveryAgent: 11.38s
    Tokens: 18346 in, 418 out
    Tools: aap_list_collections: 1, aap_search_collections: 2
    collections_found: 0
  PlanningAgent: 43.29s
    Tokens: 97099 in, 2187 out
    Tools: add_checklist_task: 10, list_checklist_tasks: 2, list_directory: 2, read_file: 2
  WriteAgent: 151.61s
    Tokens: 605487 in, 8288 out
    Tools: ansible_doc_lookup: 3, ansible_lint: 2, ansible_write: 12, get_checklist_summary: 1, list_checklist_tasks: 1, read_file: 2, update_checklist_task: 10, write_file: 3
    attempts: 1
    complete: True
    files_created: 11
    files_total: 11
  ValidationAgent: 24.30s
    collections_installed: 2
    collections_failed: 0
    validators_passed: ['ansible-lint', 'role-check']
    validators_failed: []
    attempts: 0
    complete: True
    has_errors: False