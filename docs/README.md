## Semaphore Service

Declarative Ansible Semaphore 2.16 service definition that reuses the shared infrastructure adapters to deploy across Proxmox LXC, Docker Compose, Podman Quadlet, Kubernetes, or bare-metal systemd from the same contract.

### Runtime Coverage
- Proxmox LXC managed through `common.render_runtime`/`common.apply_runtime`
- Docker Compose v2 (env/file secrets rendered automatically)
- Podman Quadlet units
- Kubernetes Deployment, Service, Secret, and PVCs
- Bare-metal systemd unit with tmpfs mounts for runtime paths

### Dependencies
- `mariadb` (database host and port resolved from dependency exports when available).
  A dependency pre-flight waits for the MariaDB socket to accept connections before the
  runtime manifests are rendered so that ordering is enforced even when the orchestrator
  does not guarantee role sequencing.

### Exports
```
SEMAPHORE_URL={{ semaphore_external_url }}
SEMAPHORE_PORT={{ semaphore_service_port }}
SEMAPHORE_ADMIN_EMAIL={{ semaphore_admin_email }}
APP_FQDN={{ semaphore_external_hostname }}
APP_PORT={{ semaphore_service_port }}
APP_BACKEND_IP={{ service_ip }}
SEMAPHORE_WEBHOOK_URL={{ semaphore_webhook_url }}
```

These are also written to `exports.env` so downstream automation and edge devices can source the values without parsing task output.

### Secrets
- `SEMAPHORE_DB_PASS` → optional database password environment variable for backwards
  compatibility. Disabled by default.
- `SEMAPHORE_DB_PASSWORD_FILE` → password provided as a mounted secret file stored at
  `/run/secrets/semaphore/db_password` (path configurable via
  `semaphore_db_password_file_path`).
- `SEMAPHORE_ADMIN_PASSWORD_FILE` → administrator password file located at
  `/run/secrets/semaphore/admin_password` (configurable via
  `semaphore_admin_password_file_path`).
- `SEMAPHORE_ADMIN_PASSWORD` → bootstrap administrator password retained for runtime
  compatibility.

The runtime adapters call the shared `common.render_secrets` helper, so any values defined in `secrets.env` or vaulted vars are rendered into the target secret store automatically. A typical playbook maps inventory/group vars to the secret renderer like this:

```yaml
- name: Deploy Semaphore
  hosts: automation_hosts
  roles:
    - role: svc-semaphore
      vars:
        runtime: docker
        dependency_registry_file: dependency-registry.yml
        semaphore_db_password: "{{ secrets.semaphore.db_password }}"
        semaphore_admin_password: "{{ secrets.semaphore.admin_password }}"
```

Where `secrets.semaphore.*` is produced by the shared secrets renderer (for example via `common.secrets_renderer`) to keep passwords out of static vars files.

### Mounts
- Persistent: `/etc/semaphore` (config), `/var/lib/semaphore` (data store),
  `/var/lib/semaphore/jobs` (playbook history and output retention)
- Ephemeral: `/run/semaphore`, `/tmp/semaphore` tmpfs mounts applied to all runtimes (Proxmox + bare-metal benefit from generated systemd drop-ins)

### Health Checks and Probes
- Liveness: `curl -fsS http://127.0.0.1:{{ semaphore_service_port }}/`
- Readiness: `curl -fsS http://127.0.0.1:{{ semaphore_service_port }}/api/ping`
- Startup: `curl -fsS http://127.0.0.1:{{ semaphore_service_port }}/api/ping` with a 60s
  initial delay to accommodate schema migrations

### Key Overrides
| Variable | Default | Purpose |
| --- | --- | --- |
| `semaphore_service_port` | `3000` | Published HTTP port |
| `semaphore_db_host` | `mariadb` (or dependency export) | Database hostname |
| `semaphore_db_port` | `3306` | MariaDB port appended automatically when constructing URLs |
| `semaphore_db_options` | `tls=true&tls-skip-verify=false` | Enables database TLS with
verification |
| `semaphore_db_max_open_conns` | `50` | Connection pool upper bound |
| `semaphore_db_max_idle_conns` | `10` | Connection pool idle bound |
| `semaphore_config_volume_size_gb` | `2` | Persistent config volume size |
| `semaphore_data_volume_size_gb` | `20` | Persistent data volume size |
| `semaphore_jobs_volume_size_gb` | `10` | Persistent job history volume size |
| `semaphore_container_vmid` | `230` | Proxmox VMID |
| `semaphore_container_memory_mb` | `2048` | Memory reservation across runtimes |
| `semaphore_container_cpu_cores` | `2` | vCPU allocation across runtimes |
| `semaphore_kubernetes_namespace` | `automation` | Namespace for the workload |
| `semaphore_session_timeout_seconds` | `86400` | Session lifetime configurable via env |
| `semaphore_max_parallel_tasks` | `25` | Upper bound on simultaneous tasks |
| `semaphore_cleanup_days` | `30` | Automated purge window for historic jobs |
| `semaphore_cors_allowed_origins` | `` | Comma-separated list for API CORS policy |
| `semaphore_webhook_url` | `` | Optional externally reachable webhook endpoint |

### Usage
```yaml
- hosts: automation_hosts
  roles:
    - role: svc-semaphore
      vars:
        runtime: docker
        dependency_registry_file: dependency-registry.yml
        semaphore_db_password: "{{ vault_semaphore_db_password }}"
        semaphore_admin_password: "{{ vault_semaphore_admin_password }}"
```


### Reverse Proxy, TLS, and CORS
- Set `semaphore_external_url` (or `SEMAPHORE_URL`) to the public HTTPS URL that clients
  will use. The role will derive `APP_FQDN` from this value for edge integrations.
- Database connections default to TLS with verification. Toggle
  `semaphore_db_tls_enabled`/`semaphore_db_tls_skip_verify` or provide a custom
  `semaphore_db_options` string to meet security requirements.
- Terminate TLS at your reverse proxy and forward traffic to `APP_BACKEND_IP:APP_PORT`
  (defaults `{{ service_ip }}`:`{{ semaphore_service_port }}`) with the `Host` header
  preserved. If the workload must serve HTTPS directly, mount certificates into the
  runtime and adjust the proxy configuration accordingly.
- Configure the `semaphore_cors_allowed_origins` variable when the API must be invoked
  from third-party origins.
- When running behind Traefik, Caddy, or Nginx, proxy `/` to the backend and configure
  WebSocket upgrades since Semaphore streams job output over WebSockets.
- Optionally publish `SEMAPHORE_WEB_ROOT` if the service will live under a sub-path; the
  generated runtime definitions propagate this setting across Docker, Podman, Kubernetes,
  Proxmox, and bare-metal systemd targets.

### Credential Rotation
- Passwords and administrator secrets are validated during deployment. Provide strong
  values via inventory, environment, or secret store variables to satisfy the minimum
  length (12) and complexity policy.
- Database and administrator passwords are exposed as mounted files so they can be
  rotated atomically. Update the backing secret and re-run the role; Semaphore reads the
  new values during the next reconciliation.
- `semaphore_render_db_password_env` can be set to `true` when legacy environments still
  require password exposure via environment variables.
