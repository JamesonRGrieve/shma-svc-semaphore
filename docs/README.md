## Semaphore Service

Declarative Ansible Semaphore 2.16 service definition that reuses the shared infrastructure adapters to deploy across Proxmox LXC, Docker Compose, Podman Quadlet, Kubernetes, or bare-metal systemd from the same contract.

### Runtime Coverage
- Proxmox LXC managed through `common.render_runtime`/`common.apply_runtime`
- Docker Compose v2 (env/file secrets rendered automatically)
- Podman Quadlet units
- Kubernetes Deployment, Service, Secret, and PVCs
- Bare-metal systemd unit with tmpfs mounts for runtime paths

### Dependencies
- `mariadb (>=10.3)` (database host and port resolved from dependency exports when
  available). A dependency pre-flight waits for the MariaDB socket to accept connections
  before the runtime manifests are rendered so that ordering is enforced even when the
  orchestrator does not guarantee role sequencing.
- When dependency exports declare a MariaDB dependency without `DATABASE_HOST` or a
  version string, the role fails fast with an actionable error instead of silently
  falling back to environment variables. This prevents mis-routed traffic when multiple
  MariaDB clusters are present.
- `semaphore_db_verify_connectivity` performs both a TCP wait and an authenticated
  `SELECT 1` through `community.mysql.mysql_query`, catching credential issues before the
  Semaphore containers start.

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
The rendered file now uses shell-style environment fallbacks (for example,
`APP_FQDN=${SEMAPHORE_EXTERNAL_HOSTNAME:-...}`) so operators can override exports at
runtime without re-templating the contract.

### Secrets
- `SEMAPHORE_DB_PASS` → optional database password environment variable for backwards
  compatibility. Disabled by default.
- `SEMAPHORE_DB_PASSWORD_FILE` → password provided as a mounted secret file stored at
  `/run/secrets/db_password` (path configurable via
  `semaphore_db_password_file_path` and `semaphore_secret_mount_prefix`).
- `SEMAPHORE_ADMIN_PASSWORD_FILE` → administrator password file located at
  `/run/secrets/admin_password` (configurable via
  `semaphore_admin_password_file_path`).
- `SEMAPHORE_ADMIN_PASSWORD` → bootstrap administrator password retained for runtime
  compatibility.

File-based secrets remain the default because they align with Docker/Podman secret
mounts, but `semaphore_render_db_password_env: true` can re-enable the legacy environment
variable path when migrating older deployments. Document and gate that mode carefully
because the service now enforces stronger validations against weak credentials.

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
| `semaphore_estimated_daily_artifacts_mb` | `0` | Estimated artifact footprint used to warn about volume sizing |
| `semaphore_container_vmid` | `230` | Proxmox VMID |
| `semaphore_container_memory_mb` | `2048` | Memory reservation across runtimes |
| `semaphore_container_cpu_cores` | `2` | vCPU allocation across runtimes |
| `semaphore_prefer_ipv6` | `false` | Prefer host IPv6 facts when computing `service_ip` |
| `semaphore_kubernetes_namespace` | `automation` | Namespace for the workload |
| `semaphore_session_timeout_seconds` | `86400` | Session lifetime configurable via env |
| `semaphore_max_parallel_tasks` | `25` | Upper bound on simultaneous tasks |
| `semaphore_cleanup_days` | `30` | Automated purge window for historic jobs |
| `semaphore_cors_allowed_origins` | `` | Comma-separated list for API CORS policy |
| `semaphore_webhook_url` | `` | Optional externally reachable webhook endpoint |
| `semaphore_verify_admin_password` | `false` | Perform a post-deploy API login check using the configured admin credentials |

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

#### Reverse Proxy Rate Limiting Examples

Semaphore's login endpoint is susceptible to password spraying without upstream rate
limiting. The snippets below add conservative throttling while preserving legitimate
automation traffic:

```nginx
limit_req_zone $binary_remote_addr zone=semaphore_login:10m rate=30r/m;

server {
  location / {
    limit_req zone=semaphore_login burst=10 nodelay;
    proxy_pass http://{{ service_ip }}:{{ semaphore_service_port }};
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

```haproxy
frontend semaphore
    bind *:443 ssl crt /etc/haproxy/certs
    acl semaphore_login path_beg /api/auth/login
    tcp-request connection track-sc1 src
    tcp-request content reject if semaphore_login { sc_http_req_rate(0) gt 30 }
    use_backend semaphore_backend

backend semaphore_backend
    server svc {{ service_ip }}:{{ semaphore_service_port }} check
```

```yaml
## Traefik static configuration snippet
http:
  middlewares:
    semaphore-rate-limit:
      rateLimit:
        average: 30
        burst: 10
```

Attach the middleware to the entrypoint serving Semaphore to rate limit `/api/auth/login`
requests.

### Credential Rotation
- Passwords and administrator secrets are validated during deployment. Provide strong
  values via inventory, environment, or secret store variables to satisfy the minimum
  length (12 by default), any optional regex (`semaphore_password_complexity_regex`), and
  the curated `semaphore_common_weak_passwords` deny-list.
- Database and administrator passwords are exposed as mounted files so they can be
  rotated atomically. Update the backing secret and re-run the role; Semaphore reads the
  new values during the next reconciliation.
- `semaphore_render_db_password_env` can be set to `true` when legacy environments still
  require password exposure via environment variables, but prefer file-based secrets for
  new deployments.
- Set `semaphore_verify_admin_password: true` to perform a post-deploy API login using the
  configured administrator credentials as a smoke test after reconciliation.

### Webhook Integration
- `SEMAPHORE_WEBHOOK_URL` / `semaphore_webhook_url` publishes the endpoint external
  services should call to trigger Semaphore jobs. Provide a globally reachable URL when
  integrating CI/CD systems; the role emits a warning when the value resembles a private
  address such as `http://192.168.*` or `http://127.0.0.1`.
- Webhooks follow the Semaphore API contract. A minimal trigger looks like:

  ```bash
  curl -X POST "${SEMAPHORE_WEBHOOK_URL}" \
       -H 'Content-Type: application/json' \
       -d '{"event":"deploy","payload":{"environment":"prod"}}'
  ```

  Authenticate the request with the token configured inside Semaphore's UI and ensure the
  reverse proxy allows POST bodies up to the expected payload size.

### Upgrading Semaphore
- Review Semaphore release notes between your current version and `{{ version }}` for
  database schema changes. Releases prior to 2.10 used different migration paths than the
  current 2.16 series.
- Before upgrading, back up the MariaDB database and Semaphore volumes:

  ```bash
  mysqldump -h ${SEMAPHORE_DB_HOST} -u ${SEMAPHORE_DB_USER} -p${SEMAPHORE_DB_PASS} ${SEMAPHORE_DB_NAME} \
    > semaphore-$(date +%F).sql
  tar czf semaphore-volumes-$(date +%F).tgz /var/lib/semaphore /etc/semaphore
  ```

- Apply the updated role. The deployment waits for dependency health and validates
  credentials before reconciling manifests.
- If the upgrade encounters issues, restore the database dump, roll back the service
  container image, and re-run the previous role version. Keep the dumps until post-upgrade
  validation (including the optional admin login check) succeeds.
