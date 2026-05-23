# Element Web Client — Design Spec

**Date:** 2026-05-24
**Scope:** Docker deployment mode only

---

## Goal

Add an optional web client container (Element Web or Cinny) to the Docker Compose stack so users can reach the Matrix homeserver from a browser without configuring a client manually. The role generates the client's `config.json` automatically from existing Ansible variables.

---

## New Variables (`defaults/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `synapse_element_enabled` | `false` | Enable the web client service |
| `synapse_element_client` | `element` | Client to deploy: `element` or `cinny` |
| `synapse_element_port` | `8080` | Host port the web client is exposed on |
| `synapse_element_image` | `""` | Override the container image; empty = auto-select |

**Auto-selected images (resolved in `docker-compose.yml.j2`):**

| Client | Default image |
|---|---|
| `element` | `ghcr.io/element-hq/element-web:v1.11.96` |
| `cinny` | `ghcr.io/cinnyapp/cinny:v4.12.2` |

Both images are pinned to specific versions (no `latest` tags).

---

## New Templates

### `templates/element-web-config.json.j2`

Generated config for Element Web. Key fields:

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

### `templates/cinny-config.json.j2`

Generated config for Cinny. Different schema from Element — `homeserverList` is a plain string array of domain names (Cinny resolves the homeserver URL from the domain):

```json
{
  "defaultHomeserver": 0,
  "homeserverList": ["{{ synapse_server_name }}"],
  "allowCustomHomeservers": false
}
```

Both clients mount their config at `/app/config.json` inside the container.

---

## Task Changes (`tasks/docker/configure.yml`)

One new task appended, guarded by `when: synapse_element_enabled`:

```yaml
- name: Deploy web client config
  ansible.builtin.template:
    src: "{{ synapse_element_client }}-config.json.j2"
    dest: "{{ synapse_docker_data_dir }}/element-config.json"
    mode: "0644"
  when: synapse_element_enabled
  notify: Restart synapse container
```

The config file is always written to the same host path regardless of client, so the Compose volume mount is unconditional within the service block.

---

## Docker Compose Changes (`templates/docker-compose.yml.j2`)

Conditional service block appended at the end of the file:

```yaml
{% if synapse_element_enabled %}
{% set _element_images = {
  'element': 'ghcr.io/element-hq/element-web:v1.11.96',
  'cinny': 'ghcr.io/cinnyapp/cinny:v4.12.2'
} %}
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

Image resolution is done inline in Jinja: dict lookup by `synapse_element_client`, overridden by `synapse_element_image` if set.

---

## Molecule Changes

### `molecule/docker/molecule.yml`

Add to `host_vars.synapse-docker-host`:

```yaml
synapse_element_enabled: true
```

`synapse_element_client` defaults to `element` — no need to set it explicitly in the test.

### `molecule/docker/verify.yml`

Three new checks appended after the existing Synapse HTTP check:

1. Assert `synapse-element` container is running (via `community.docker.docker_container_info`)
2. Poll `http://localhost:{{ synapse_element_port }}` with `ansible.builtin.uri` until HTTP 200 (retries: 12, delay: 5s)
3. Fetch `http://localhost:{{ synapse_element_port }}/config.json` and assert it contains `synapse_server_name` — proves the generated config was picked up, not just that nginx is alive

The `uri` checks work because `synapse_element_port` is bound to the DinD host's localhost — same mechanism the existing Synapse port check relies on (just using `uri` instead of `docker exec` since Element Web is a static nginx server with no Python).

---

## What Is Not In Scope

- Server mode (`synapse_deployment_mode: server`) — Element Web is Docker-only in this role
- Reverse proxy / TLS for Element Web — out of scope; users deploy their own proxy
- Authentication testing (actually logging in via Matrix API) — the verify test checks reachability only
- Molecule `default` scenario changes — server scenario is unaffected

---

## File Changeset Summary

| File | Change |
|---|---|
| `defaults/main.yml` | Add 4 new vars |
| `templates/element-web-config.json.j2` | New file |
| `templates/cinny-config.json.j2` | New file |
| `tasks/docker/configure.yml` | Add 1 task |
| `templates/docker-compose.yml.j2` | Add conditional service block |
| `molecule/docker/molecule.yml` | Add `synapse_element_enabled: true` |
| `molecule/docker/verify.yml` | Add 2 verification tasks |
