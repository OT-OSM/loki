[![Apache License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
![GitHub release (latest by date)](https://img.shields.io/github/v/release/OT-OSM/loki)
[![Ansible](https://img.shields.io/badge/Ansible-Role-red.svg)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
[![Platform](https://img.shields.io/badge/Platform-Ubuntu%2020.04%20%7C%2022.04-orange.svg)](https://ubuntu.com)

[![Opstree Solutions][opstree_avatar]][opstree_homepage]<br/>[Opstree Solutions][opstree_homepage]

[opstree_homepage]: https://opstree.github.io/
[opstree_avatar]: https://img.cloudposse.com/150x150/https://github.com/opstree.png

---

# Loki — Ansible Role

A production-grade Ansible role to install, configure, and manage **Grafana Loki** on Ubuntu systems. Loki is a horizontally scalable, highly available log aggregation system inspired by Prometheus. It indexes only metadata (labels) rather than full log content, making it significantly more cost-efficient than traditional log aggregation solutions. Loki integrates natively with Grafana and is designed to work alongside Tempo and Prometheus in a full observability stack.

## Key Features

- [x] Installs Loki from official Grafana GitHub releases
- [x] Supports architecture-specific binary selection (e.g. `linux_amd64`, `linux_arm64`)
- [x] Creates a dedicated system user and group for security isolation
- [x] Configures Loki via Jinja2 templates
- [x] Manages service lifecycle using Ansible handlers
- [x] Idempotent — safe to re-run without side effects
- [x] All variables are role-namespaced to avoid conflicts

---

## Requirements

| Requirement | Details |
|-------------|---------|
| **OS** | Ubuntu `focal` (20.04) or `jammy` (22.04) |
| **Privileges** | Root or sudo access on target hosts |
| **Ansible Collection** | `community.general` |

Install the required collection:

```bash
ansible-galaxy collection install community.general
```

> **Security Note:** Sensitive defaults are placeholders in `defaults/main.yml`.  
> Always override secrets via **Semaphore environment variables** or **Ansible Vault** — never commit credentials to source control.

---

## Role Structure

```
loki/
├── vars/
│   └── main.yml              # Internal role variables (higher precedence)
├── tasks/
│   └── main.yml              # Main task entry point
├── handlers/
│   └── main.yml              # Service restart / reload handlers
└── templates/
    ├── loki-config.yaml.j2   # Main Loki configuration template
    └── loki.service.j2       # Jinja2 systemd unit template
```

---

## File Descriptions

### `tasks/main.yml`

Orchestrates all installation and configuration steps:

1. Create dedicated `loki` system user and group
2. Create data, WAL, chunks, and configuration directories with correct ownership
3. Download the Loki binary archive from Grafana GitHub releases for the target architecture
4. Extract and install the binary to the system PATH
5. Render and deploy the Loki config and systemd unit from templates
6. Enable and start the `loki` service

### `handlers/main.yml`

Triggered automatically when configuration changes are detected:

- **`restart loki`** — restarts the Loki service
- **`reload systemd`** — reloads the systemd daemon after service file updates

### `templates/loki-config.yaml.j2`

Jinja2 template that renders the main Loki configuration installed to `/etc/loki/loki-config.yaml`. Defines the server, ingester, querier, compactor, schema config, storage config, and limits sections using role variables.

### `templates/loki.service.j2`

Jinja2 template that renders the systemd unit file installed to `/etc/systemd/system/loki.service`. References the configuration file path and runs Loki under the dedicated service user.

---

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `loki_version` | `3.0.0` | Version of Loki to install |
| `loki_arch` | `linux_amd64` | Target architecture for the binary download (`linux_amd64`, `linux_arm64`) |
| `loki_user` | `loki` | System user that runs the service |
| `loki_group` | `loki` | System group for the service user |
| `loki_http_port` | `3100` | Port for the Loki HTTP API and health endpoint |
| `loki_grpc_port` | `9096` | Port for the Loki gRPC API |
| `loki_config_dir` | `/etc/loki` | Directory for configuration files |
| `loki_data_dir` | `/var/lib/loki` | Directory for persistent chunks, index, and WAL data |
| `loki_install_dir` | `/usr/local/bin` | Directory for the installed binary |
| `loki_storage_backend` | `filesystem` | Storage backend to use (`filesystem`, `s3`, `gcs`, `azure`) |
| `loki_retention_period` | `744h` | How long to retain log data (default 31 days) |
| `loki_ingestion_rate_mb` | `4` | Per-tenant ingestion rate limit in MB/s |
| `loki_ingestion_burst_size_mb` | `6` | Per-tenant ingestion burst size limit in MB |

> Override any variable in your playbook, inventory, or via `--extra-vars`.

---

## Usage

### Quick Start

```bash
ansible-playbook -i inventory playbook.yml
```

### Run Specific Phases

```bash
# Install binary only
ansible-playbook -i inventory playbook.yml --tags install

# Configure service only
ansible-playbook -i inventory playbook.yml --tags configure

# Restart service only
ansible-playbook -i inventory playbook.yml --tags service
```

### Example Playbook

```yaml
---
- name: Deploy Grafana Loki
  hosts: log_servers
  become: true
  roles:
    - role: loki
      vars:
        loki_version: "3.0.0"
        loki_arch: "linux_amd64"
        loki_user: "loki"
        loki_group: "loki"
        loki_http_port: 3100
        loki_storage_backend: "filesystem"
        loki_retention_period: "744h"
```

---

## Tags

| Tag | Description |
|-----|-------------|
| `install` | Download and install the Loki binary |
| `configure` | Render and deploy all configuration templates |
| `service` | Start, stop, or restart the service |

---

## Handlers

| Handler | Trigger Condition | Action |
|---------|------------------|--------|
| `restart loki` | Config or binary change | Restarts the Loki service |
| `reload systemd` | Service unit file updated | Reloads the systemd daemon |

---

## Templates

| Template | Destination | Description |
|----------|-------------|-------------|
| `loki-config.yaml.j2` | `/etc/loki/loki-config.yaml` | Main Loki pipeline configuration (server, ingester, storage, schema, limits) |
| `loki.service.j2` | `/etc/systemd/system/loki.service` | Systemd service definition |

---

## Verification

After running the playbook, confirm Loki is running correctly.

### Check service status

```bash
systemctl status loki
```

### Verify the HTTP API is reachable

```bash
curl -s http://localhost:3100/ready
```

Expected output:

```
ready
```

### Check overall health

```bash
curl -s http://localhost:3100/loki/api/v1/status/buildinfo
```

Expected output (JSON):

```json
{
  "version": "3.0.0",
  "branch": "HEAD",
  "buildDate": "...",
  "goVersion": "..."
}
```

### Query ingested labels

```bash
curl -s 'http://localhost:3100/loki/api/v1/labels'
```

Expected output:

```json
{
  "status": "success",
  "data": ["job", "host", "level"]
}
```

### Confirm listening ports

```bash
ss -tulpn | grep loki
```

Expected output:

```
LISTEN   0   4096   *:3100   *:*   users:(("loki",pid=XXXX,fd=3))
LISTEN   0   4096   *:9096   *:*   users:(("loki",pid=XXXX,fd=4))
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Service fails to start | Config YAML syntax error | Run `loki -config.file=/etc/loki/loki-config.yaml` manually to see the error |
| Port 3100 already in use | Another process bound to the port | Change `loki_http_port` or stop the conflicting process |
| Logs not appearing in Grafana | Grafana datasource URL wrong | Set the Loki datasource URL to `http://<host>:3100` in Grafana |
| Data directory permission error | Wrong ownership on data dir | Ensure `loki_data_dir` is owned by `loki_user` |
| Ingestion rate limit errors | Default limits too low for volume | Increase `loki_ingestion_rate_mb` and `loki_ingestion_burst_size_mb` |
| High disk usage | Retention period too long or compaction not running | Reduce `loki_retention_period` and verify compactor is enabled in config |
| Binary not found after install | Wrong architecture selected | Verify `loki_arch` matches the target host (`uname -m`) |
| `systemctl` not found | Non-systemd system | This role requires systemd — not supported on older init systems |

---

## References

| Resource | Link |
|----------|------|
| Grafana Loki Official Documentation | https://grafana.com/docs/loki/latest/ |
| Loki GitHub | https://github.com/grafana/loki |
| Loki Configuration Reference | https://grafana.com/docs/loki/latest/configure/ |
| Loki LogQL Query Language | https://grafana.com/docs/loki/latest/query/ |
| Ansible Roles Documentation | https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html |
| Ansible Handlers Documentation | https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html |

---

## Authors

| Name | Email | Organization |
|------|-------|-------------|
| Abhishek Vishwakarma | abhishek.vishwakarma@opstree.com | Opstree Solutions |
| Shubham Rathi | shubham.rathi@mygurukulam.co | MyGurukulam |

---

## License

This project is licensed under the [Apache 2.0 License](LICENSE).
