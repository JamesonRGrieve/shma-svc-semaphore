## Semaphore Service

Declarative Ansible Semaphore 2.16 service definition that reuses the shared infrastructure adapters to deploy across Proxmox LXC, Docker Compose, Podman Quadlet, Kubernetes, or bare-metal systemd from the same contract.

### Runtime Coverage
- Proxmox LXC managed through `common.render_runtime`/`common.apply_runtime`
- Docker Compose v2 (env/file secrets rendered automatically)
- Podman Quadlet units
- Kubernetes Deployment, Service, Secret, and PVCs
- Bare-metal systemd unit with tmpfs mounts for runtime paths

### Dependencies
- `mariadb` (database host and port resolved from dependency exports when available)

### Exports
```
SEMAPHORE_URL={{ semaphore_external_url }}
SEMAPHORE_PORT={{ semaphore_service_port }}
SEMAPHORE_ADMIN_EMAIL={{ semaphore_admin_email }}
APP_FQDN={{ semaphore_external_hostname }}
APP_PORT={{ semaphore_service_port }}
APP_BACKEND_IP={{ service_ip }}
```

These are also written to `exports.env` so downstream automation and edge devices can source the values without parsing task output.

### Secrets
- `SEMAPHORE_DB_PASS` → database password forwarded to the Semaphore container
- `SEMAPHORE_ADMIN_PASSWORD` → bootstrap administrator password

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
- Persistent: `/etc/semaphore` (config), `/var/lib/semaphore` (data store)
- Ephemeral: `/run/semaphore`, `/tmp/semaphore` tmpfs mounts applied to all runtimes (Proxmox + bare-metal benefit from generated systemd drop-ins)

### Health Check
`curl -fsS http://127.0.0.1:{{ semaphore_service_port }}/` expecting the login page; reused for Compose, Quadlet, Kubernetes, and the deployment gate.

### Key Overrides
| Variable | Default | Purpose |
| --- | --- | --- |
| `semaphore_service_port` | `3000` | Published HTTP port |
| `semaphore_db_host` | `mariadb` (or dependency export) | Database hostname |
| `semaphore_db_user` | `semaphore` | Database username |
| `semaphore_config_volume_size_gb` | `2` | Persistent config volume size |
| `semaphore_data_volume_size_gb` | `20` | Persistent data volume size |
| `semaphore_container_vmid` | `230` | Proxmox VMID |
| `semaphore_container_memory_mb` | `2048` | Memory reservation across runtimes |
| `semaphore_container_cpu_cores` | `2` | vCPU allocation across runtimes |
| `semaphore_kubernetes_namespace` | `automation` | Namespace for the workload |

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


### Reverse Proxy and TLS
- Set `semaphore_external_url` (or `SEMAPHORE_URL`) to the public HTTPS URL that clients will use. The role will derive `APP_FQDN` from this value for edge integrations.
- Terminate TLS at your reverse proxy and forward traffic to `APP_BACKEND_IP:APP_PORT` (defaults `{{ service_ip }}`:`{{ semaphore_service_port }}`) with the `Host` header preserved.
- When running behind Traefik, Caddy, or Nginx, proxy `/` to the backend and configure WebSocket upgrades since Semaphore streams job output over WebSockets.
- Optionally publish `SEMAPHORE_WEB_ROOT` if the service will live under a sub-path; the generated runtime definitions propagate this setting across Docker, Podman, Kubernetes, Proxmox, and bare-metal systemd targets.
