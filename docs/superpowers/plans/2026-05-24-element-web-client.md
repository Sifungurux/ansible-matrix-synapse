# Element Web Client Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an optional Element Web (or Cinny) container to the Docker Compose stack with auto-generated config, and verify it with Molecule tests.

**Architecture:** Four role files change — defaults, two new config templates, the docker configure task, and the docker-compose template. The Molecule docker scenario enables the feature and gains three verify checks (container running, HTTP 200, config.json contains server name).

**Tech Stack:** Ansible, Jinja2, Molecule, Docker Compose v2, Element Web (nginx-alpine), Cinny (nginx-alpine)

---

## File Map

| File | Action |
|---|---|
| `defaults/main.yml` | Modify — add 4 new variables |
| `templates/element-web-config.json.j2` | Create — Element Web config template |
| `templates/cinny-config.json.j2` | Create — Cinny config template |
| `tasks/docker/configure.yml` | Modify — add config deploy task before docker-compose deploy |
| `templates/docker-compose.yml.j2` | Modify — add conditional element service block |
| `molecule/docker/molecule.yml` | Modify — enable `synapse_element_enabled: true` |
| `molecule/docker/verify.yml` | Modify — add 3 element verification tasks |

---

## Task 1: Create a feature branch

- [ ] **Step 1: Create and switch to the feature branch**

```bash
git checkout -b feature/element-web-client
```

Expected output:
```
Switched to a new branch 'feature/element-web-client'
```

---

## Task 2: Write the failing Molecule tests (TDD — tests first)

**Files:**
- Modify: `molecule/docker/molecule.yml`
- Modify: `molecule/docker/verify.yml`

- [ ] **Step 1: Enable element in the Molecule scenario**

In `molecule/docker/molecule.yml`, add `synapse_element_enabled: true` to `host_vars.synapse-docker-host`. The full `host_vars` block after the change:

```yaml
      synapse-docker-host:
        synapse_server_name: test.matrix.local
        synapse_deployment_mode: docker
        synapse_database_engine: postgresql
        synapse_database_password: synapse-test-password
        synapse_tls_mode: none
        synapse_registration_shared_secret: test-secret-change-me
        synapse_create_admin: false
        synapse_element_enabled: true
```

- [ ] **Step 2: Add element verification tasks to `molecule/docker/verify.yml`**

Append the following three tasks after the existing `Assert response contains versions` task. The full file after the change:

```yaml
---
- name: Verify
  hosts: all
  become: true
  tasks:
    - name: Check Synapse container is running
      community.docker.docker_container_info:
        name: synapse
      register: _synapse_container

    - name: Assert container is running
      ansible.builtin.assert:
        that:
          - _synapse_container.container.State.Running

    - name: Check PostgreSQL container is running
      community.docker.docker_container_info:
        name: synapse-postgres
      register: _postgres_container

    - name: Assert PostgreSQL is running
      ansible.builtin.assert:
        that:
          - _postgres_container.container.State.Running

    - name: Collect Synapse container logs
      ansible.builtin.command:
        cmd: docker logs synapse --tail 80
      register: _synapse_logs
      changed_when: false
      failed_when: false

    - name: Show Synapse container logs
      ansible.builtin.debug:
        msg: "{{ (_synapse_logs.stdout_lines | default([])) + (_synapse_logs.stderr_lines | default([])) }}"

    - name: Check Synapse responds on HTTP port
      ansible.builtin.command:
        cmd: >-
          docker exec synapse python3 -c
          "import urllib.request;
          print(urllib.request.urlopen(
          'http://localhost:8008/_matrix/client/versions').read().decode())"
      register: _synapse_versions
      retries: 24
      delay: 5
      until: _synapse_versions.rc == 0
      changed_when: false

    - name: Assert response contains versions
      ansible.builtin.assert:
        that:
          - "'versions' in _synapse_versions.stdout"

    - name: Check Element Web container is running
      community.docker.docker_container_info:
        name: synapse-element
      register: _element_container

    - name: Assert Element Web container is running
      ansible.builtin.assert:
        that:
          - _element_container.container.State.Running

    - name: Check Element Web responds on HTTP port
      ansible.builtin.command:
        cmd: docker exec synapse-element wget -qO- http://localhost/
      register: _element_response
      retries: 12
      delay: 5
      until: _element_response.rc == 0
      changed_when: false

    - name: Check Element Web config.json contains server name
      ansible.builtin.command:
        cmd: docker exec synapse-element wget -qO- http://localhost/config.json
      register: _element_config
      retries: 6
      delay: 5
      until: _element_config.rc == 0
      changed_when: false

    - name: Assert config.json references the homeserver
      ansible.builtin.assert:
        that:
          - "'test.matrix.local' in _element_config.stdout"
```

- [ ] **Step 3: Commit the failing tests**

```bash
git add molecule/docker/molecule.yml molecule/docker/verify.yml
git commit -m "test: add Molecule verify checks for Element Web container"
```

---

## Task 3: Add defaults variables

**Files:**
- Modify: `defaults/main.yml`

- [ ] **Step 1: Append the four new variables to `defaults/main.yml`**

Add at the end of the file, after the `synapse_extra_config: {}` line:

```yaml

# Web client (docker mode only)
synapse_element_enabled: false
synapse_element_client: element
synapse_element_port: 8080
synapse_element_image: ""
```

- [ ] **Step 2: Commit**

```bash
git add defaults/main.yml
git commit -m "feat: add synapse_element_* defaults variables"
```

---

## Task 4: Create config templates

**Files:**
- Create: `templates/element-web-config.json.j2`
- Create: `templates/cinny-config.json.j2`

- [ ] **Step 1: Create `templates/element-web-config.json.j2`**

```json
{
  "default_server_config": {
    "m.homeserver": {
      "base_url": "{{ synapse_public_baseurl }}",
      "server_name": "{{ synapse_server_name }}"
    }
  },
  "brand": "Element",
  "disable_guests": true
}
```

- [ ] **Step 2: Create `templates/cinny-config.json.j2`**

Cinny's `homeserverList` is a plain string array of domain names — no `{name, url}` objects:

```json
{
  "defaultHomeserver": 0,
  "homeserverList": ["{{ synapse_server_name }}"],
  "allowCustomHomeservers": false
}
```

- [ ] **Step 3: Commit**

```bash
git add templates/element-web-config.json.j2 templates/cinny-config.json.j2
git commit -m "feat: add Element Web and Cinny config.json templates"
```

---

## Task 5: Add the config deploy task to `tasks/docker/configure.yml`

**Files:**
- Modify: `tasks/docker/configure.yml`

The new task must be inserted **before** the `Deploy docker-compose.yml` task so the config file exists on disk when Compose starts the element container for the first time.

- [ ] **Step 1: Insert the new task in `tasks/docker/configure.yml`**

The full file after the change:

```yaml
---
- name: Ensure Docker project directory exists
  ansible.builtin.file:
    path: "{{ synapse_docker_compose_dir }}"
    state: directory
    mode: "0755"

- name: Ensure Docker data directory exists
  ansible.builtin.file:
    path: "{{ synapse_docker_data_dir }}"
    state: directory
    mode: "0777"

- name: Deploy homeserver.yaml for Docker
  ansible.builtin.template:
    src: homeserver.yaml.j2
    dest: "{{ synapse_docker_data_dir }}/homeserver.yaml"
    mode: "0644"
  vars:
    synapse_config_dir: /data
    synapse_data_dir: /data
    synapse_media_store_path: /data/media_store
    synapse_signing_key_path: /data/homeserver.signing.key
    synapse_log_dir: /data
    synapse_pid_file: /data/homeserver.pid
    synapse_database_host: >-
      {{ (synapse_database_engine == 'postgresql') | ternary('postgres', 'localhost') }}
  notify: Restart synapse container

- name: Deploy log.yaml for Docker
  ansible.builtin.template:
    src: log.yaml.j2
    dest: "{{ synapse_docker_data_dir }}/log.yaml"
    mode: "0644"
  vars:
    synapse_log_dir: /data
    synapse_config_dir: /data
  notify: Restart synapse container

- name: Deploy web client config
  ansible.builtin.template:
    src: "{{ synapse_element_client }}-config.json.j2"
    dest: "{{ synapse_docker_data_dir }}/element-config.json"
    mode: "0644"
  when: synapse_element_enabled
  notify: Restart synapse container

- name: Deploy docker-compose.yml
  ansible.builtin.template:
    src: docker-compose.yml.j2
    dest: "{{ synapse_docker_compose_dir }}/docker-compose.yml"
    mode: "0640"
  notify: Restart synapse container

- name: Start Synapse containers
  community.docker.docker_compose_v2:
    project_src: "{{ synapse_docker_compose_dir }}"
    state: present
  changed_when: false
```

- [ ] **Step 2: Commit**

```bash
git add tasks/docker/configure.yml
git commit -m "feat: deploy web client config.json before docker-compose starts"
```

---

## Task 6: Add the element service to `templates/docker-compose.yml.j2`

**Files:**
- Modify: `templates/docker-compose.yml.j2`

The full file after the change:

```yaml
# {{ ansible_managed }}
services:
  synapse:
    image: {{ synapse_docker_image }}
    container_name: synapse
    restart: unless-stopped
    volumes:
      - {{ synapse_docker_data_dir }}:/data
    ports:
      - "{{ synapse_http_port }}:8008"
{% if synapse_tls_mode != 'none' %}
      - "{{ synapse_federation_port }}:8448"
{% endif %}
    environment:
      SYNAPSE_CONFIG_PATH: /data/homeserver.yaml
{% if synapse_database_engine == 'postgresql' %}
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16.13-alpine
    container_name: synapse-postgres
    restart: unless-stopped
    volumes:
      - {{ synapse_docker_data_dir }}/postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: {{ synapse_database_user }}
      POSTGRES_PASSWORD: {{ synapse_database_password }}
      POSTGRES_DB: {{ synapse_database_name }}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ synapse_database_user }}"]
      interval: 5s
      timeout: 5s
      retries: 5
{% endif %}
{% if synapse_element_enabled %}
{% set _element_images = {'element': 'ghcr.io/element-hq/element-web:v1.11.96', 'cinny': 'ghcr.io/cinnyapp/cinny:v4.12.2'} %}
{% set _element_image = synapse_element_image if synapse_element_image else _element_images[synapse_element_client] %}

  element:
    image: {{ _element_image }}
    container_name: synapse-element
    restart: unless-stopped
    ports:
      - "{{ synapse_element_port }}:80"
    volumes:
      - {{ synapse_docker_data_dir }}/element-config.json:/app/config.json:ro
{% endif %}
```

- [ ] **Step 1: Replace `templates/docker-compose.yml.j2` with the content above**

- [ ] **Step 2: Run yamllint to check the template**

```bash
yamllint templates/docker-compose.yml.j2
```

Expected: no errors (warnings about line length are OK — max is 200)

- [ ] **Step 3: Commit**

```bash
git add templates/docker-compose.yml.j2
git commit -m "feat: add optional element web client service to docker-compose"
```

---

## Task 7: Run the full Molecule test suite

- [ ] **Step 1: Run lint first**

```bash
yamllint . && ansible-lint
```

Expected: clean (no errors)

- [ ] **Step 2: Run the full Molecule docker scenario**

```bash
molecule test -s docker
```

Expected output (key lines):
```
PLAY [Verify] ******************************************************************
...
TASK [Assert Element Web container is running] *********************************
ok: [synapse-docker-host]
...
TASK [Assert config.json references the homeserver] ****************************
ok: [synapse-docker-host]
...
INFO     Verifier completed successfully.
INFO     Running docker > destroy
...
INFO     Scenario 'docker' test matrix: dependency, cleanup, destroy, syntax, create, prepare, converge, idempotency, side_effect, verify, cleanup, destroy
```

If `molecule test` fails at the verify step, collect logs:

```bash
molecule converge -s docker
molecule verify -s docker
molecule login -s docker   # to shell into the platform and debug
```

- [ ] **Step 3: Commit if all green**

```bash
git add .
git status   # should be clean — nothing untracked
```

No commit needed here if all previous commits are clean.

---

## Task 8: Open a pull request

- [ ] **Step 1: Push the branch**

```bash
git push -u origin feature/element-web-client
```

- [ ] **Step 2: Create the PR**

```bash
gh pr create \
  --title "Add optional Element Web / Cinny web client service" \
  --body "$(cat <<'EOF'
## Summary
- Adds optional `element` service to the Docker Compose stack (controlled by `synapse_element_enabled`)
- Supports `element` (default) or `cinny` via `synapse_element_client`; image auto-selected, overridable via `synapse_element_image`
- Role generates client-specific `config.json` from `synapse_server_name` / `synapse_public_baseurl`
- Molecule docker scenario now enables Element Web and verifies: container running, HTTP 200, config.json contains server name

## Test plan
- [ ] `yamllint . && ansible-lint` passes
- [ ] `molecule test -s docker` passes end-to-end
- [ ] `molecule test -s default` (server scenario) still passes — unaffected
EOF
)"
```
