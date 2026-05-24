# ansible-matrix-synapse

Ansible role to deploy
[Matrix Synapse](https://github.com/element-hq/synapse) homeserver.

Supports two deployment modes:

- **Server** — bare metal / VM install on Debian Bookworm
- **Docker** — containerized deployment using the official Alpine image

## Requirements

- Ansible >= 2.15
- Target: Debian 12 (Bookworm)
- Required collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

Collections: `community.postgresql`, `community.docker`, `community.crypto`

## Quick Start

### Server deployment (Debian Bookworm)

```yaml
- hosts: matrix
  become: true
  roles:
    - role: ansible-matrix-synapse
      vars:
        synapse_server_name: matrix.example.com
        synapse_deployment_mode: server
        synapse_database_engine: postgresql
        synapse_database_password: "secure-db-password"
        synapse_registration_shared_secret: "change-me"
        synapse_tls_mode: letsencrypt
        synapse_letsencrypt_email: admin@example.com
```

### Docker deployment (Alpine)

```yaml
- hosts: docker-host
  become: true
  roles:
    - role: ansible-matrix-synapse
      vars:
        synapse_server_name: matrix.example.com
        synapse_deployment_mode: docker
        synapse_database_engine: postgresql
        synapse_database_password: "secure-db-password"
        synapse_registration_shared_secret: "change-me"
        synapse_tls_mode: none  # Use a reverse proxy for TLS
```

## Role Variables

See `defaults/main.yml` for all defaults.

| Variable | Description |
|----------|-------------|
| `synapse_deployment_mode` | `server` or `docker` |
| `synapse_server_name` | Server name — **immutable after first run** |
| `synapse_public_baseurl` | Public base URL |
| `synapse_version` | Docker image tag |
| `synapse_tls_mode` | `letsencrypt`, `selfsigned`, or `none` |
| `synapse_letsencrypt_email` | Email for Let's Encrypt certificates |
| `synapse_database_engine` | `postgresql` (default) or `sqlite` |
| `synapse_database_host` | PostgreSQL host |
| `synapse_database_port` | PostgreSQL port |
| `synapse_database_name` | Database name |
| `synapse_database_user` | Database user |
| `synapse_database_password` | Database password |
| `synapse_http_port` | HTTP listener port |
| `synapse_federation_port` | Federation port |
| `synapse_enable_registration` | Allow open registration |
| `synapse_registration_shared_secret` | Shared secret for registration |
| `synapse_report_stats` | Report anonymous usage stats |
| `synapse_create_admin` | Create an admin user on deploy |
| `synapse_admin_user` | Admin username |
| `synapse_admin_password` | Admin password |
| `synapse_max_upload_size` | Max file upload size |
| `synapse_log_level` | Log level |
| `synapse_turn_uris` | TURN server URIs |
| `synapse_turn_shared_secret` | TURN shared secret |
| `synapse_extra_config` | Extra keys merged into homeserver.yaml |

### Web client (docker mode only)

| Variable | Default | Description |
|----------|---------|-------------|
| `synapse_element_enabled` | `false` | Deploy a web client container alongside Synapse |
| `synapse_element_client` | `element` | Client to deploy: `element` or `cinny` |
| `synapse_element_port` | `8080` | Host port the web client listens on |
| `synapse_element_image` | `""` | Override the container image; leave empty to use the pinned default |

Default images (pinned):

- Element Web: `ghcr.io/element-hq/element-web:v1.11.96`
- Cinny: `ghcr.io/cinnyapp/cinny:v4.12.2`

## TLS Options

### Let's Encrypt (default, recommended for production)

```yaml
synapse_tls_mode: letsencrypt
synapse_letsencrypt_email: admin@example.com
synapse_letsencrypt_domain: matrix.example.com
```

### Self-signed (development/testing)

```yaml
synapse_tls_mode: selfsigned
```

### None (behind reverse proxy)

```yaml
synapse_tls_mode: none
```

## Local Testing with Lima

A [Lima](https://lima-vm.io/) VM config is included for running the role locally against a real Debian 12 host with Docker pre-installed. Ports 8008 (Synapse) and 8080 (Element Web) are forwarded to `localhost`. The provision script installs Docker Engine and adds the Lima user to the `docker` group automatically — no manual steps needed after `limactl start`.

**Requirements:** Lima installed (`brew install lima`)

```bash
# 1. Create and start the VM (first run downloads the Debian image and installs Docker — takes a few minutes)
limactl create --name matrix-synapse lima/matrix-synapse.yaml
limactl start matrix-synapse

# 2. Get the SSH port
limactl show-ssh --format=config matrix-synapse
# Look for: Port NNNNN

# 3. Create your inventory (gitignored)
cp inventory.ini.example inventory.ini
# Edit inventory.ini — replace NNNNN with the port from step 2

# 4. Install collections and run the playbook
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory.ini site.yml
```

Once the playbook completes, Element Web is available at `http://localhost:8080`.

> **`synapse_public_baseurl` must be `http://localhost:8008` for local Lima testing.** This value ends up in both `homeserver.yaml` and the Element Web `config.json`. If set to a hostname that the browser cannot resolve (e.g. `matrix.local`), Element Web will load but render a blank page. For a real deployment with a public domain this is not an issue.

**Teardown:**

```bash
limactl stop matrix-synapse
limactl delete matrix-synapse
```

## Molecule Testing

This role uses [Molecule](https://molecule.readthedocs.io/) for CI testing:

```bash
# Install test dependencies
pip install molecule molecule-plugins[docker] ansible docker

# Install required collections
ansible-galaxy collection install -r requirements.yml

# Run server scenario
molecule test -s default

# Run Docker scenario
molecule test -s docker
```

> **Note:** Molecule tests run inside a Docker container with the host Docker socket mounted. Running Molecule locally on Apple Silicon (ARM) may fail due to the x86 container image used — CI (x86) is the source of truth.

## License

Apache-2.0
