# PDS Infrastructure

Ansible playbook for deploying a [Bluesky PDS](https://github.com/bluesky-social/pds) with a full observability stack (Grafana, Loki, Prometheus, Alertmanager).

## What's included

- **PDS** — Bluesky Personal Data Server via the official installer, with Ansible-managed config
- **Caddy** — Reverse proxy with automatic TLS (including wildcard certs for handle domains)
- **Watchtower** — Automatic container image updates
- **Grafana** — Dashboards for PDS overview and system metrics
- **Loki + Promtail** — Log aggregation from Docker containers
- **Prometheus + node_exporter** — System and application metrics
- **Alertmanager** — Alerts routed to Discord by default

## System requirements

- **CPU** — 1 core minimum, 2+ recommended
- **RAM** — 4 GB is plenty for most small PDSes (the playbook creates a 4 GB swap file by default as a safety net)
- **Disk** — 20 GB minimum; actual usage depends on how many accounts you host and how much media they upload
- **OS** — Debian-based Linux distro (Ubuntu, Debian, etc.)

## Prerequisites

- A server meeting the system requirements above
- A domain pointed at the server (with a wildcard `*.` subdomain for handles)
- Ansible installed locally
- A Discord webhook URL (for default alerts)

## Setup

### 1. Configure Ansible and inventory

```bash
cp ansible.cfg.example ansible.cfg
cp inventory.yml.example inventory.yml
```

Edit `ansible.cfg` to set your preferred `remote_user`, then edit `inventory.yml` with your server's IP.

### 2. Create the vault password

```bash
# Generate a random vault password
openssl rand -base64 32 > .vault-pass
chmod 600 .vault-pass
```

### 3. Configure variables

```bash
cp vars/main.yml.example vars/main.yml
```

Edit `vars/main.yml` to set your hostname, email, and any other preferences:

| Variable                  | Description                  | Default              |
| ------------------------- | ---------------------------- | -------------------- |
| `pds_data_dir`            | PDS data directory path      | `/pds`               |
| `pds_hostname`            | PDS domain name              | —                    |
| `pds_admin_email`         | Admin contact email          | —                    |
| `pds_image_tag`           | PDS Docker image tag         | `0.4`                |
| `pds_blob_upload_limit`   | Max blob upload size (bytes) | `104857600` (100 MB) |
| `pds_blobstore_type`      | Blob storage backend         | `disk`               |
| `pds_invite_required`     | Require invite codes         | `true`               |
| `pds_rate_limits_enabled` | Enable rate limiting         | `true`               |
| `grafana_hostname`        | Grafana domain name          | —                    |
| `swap_enabled`            | Create and enable swap file  | `true`               |
| `swap_size`               | Swap file size               | `4G`                 |
| `swap_swappiness`         | Kernel swappiness value      | `10`                 |

### 4. Create the secrets file

```bash
cp vars/secrets.yml.example vars/secrets.yml
```

Edit `vars/secrets.yml` and fill in all values, then encrypt it:

```bash
ansible-vault encrypt vars/secrets.yml --vault-password-file .vault-pass
```

If you're deploying to an existing server that already has a PDS, pull the secrets from it first:

```bash
ssh root@<server-ip> 'cat /pds/pds.env'  # adjust path if using a custom pds_data_dir
```

Copy `PDS_JWT_SECRET`, `PDS_ADMIN_PASSWORD`, and `PDS_PLC_ROTATION_KEY_K256_PRIVATE_KEY_HEX` into the corresponding `vault_pds_*` variables.

### 5. Deploy

```bash
ansible-playbook playbook.yml --vault-password-file .vault-pass
```

## How it works

On **first deploy**, the playbook runs the [official PDS installer script](https://github.com/bluesky-social/pds/blob/main/installer.sh), which installs Docker and sets up the initial PDS database. This step is skipped on subsequent runs (guarded by the existence of `<pds_data_dir>/account.sqlite`).

After the installer, Ansible takes over managing the configuration files:

- `<pds_data_dir>/pds.env` — PDS environment variables
- `<pds_data_dir>/compose.yaml` — Docker Compose for PDS, Caddy, and Watchtower
- `<pds_data_dir>/caddy/etc/caddy/Caddyfile` — Caddy reverse proxy config
- `/etc/systemd/system/pds.service` — Systemd unit for the PDS stack

The observability stack runs as a separate Docker Compose file (`docker-compose.observability.yml`) managed by its own systemd unit, ordered to start after the PDS.

## Editing secrets

```bash
ansible-vault edit vars/secrets.yml --vault-password-file .vault-pass
```

## Reference

### Playbook

`playbook.yml` is the single entrypoint. It runs against the `pds` host group and does everything in order:

1. **PDS setup** — Runs the official Bluesky installer on first deploy, then templates `pds.env`, `compose.yaml`, and the `pds.service` systemd unit. Changes to any of these trigger a PDS restart via handler.
2. **Swap** — Creates a 4 GB swap file with swappiness tuned to 10.
3. **Observability** — Deploys the full monitoring stack (Grafana, Loki, Promtail, Prometheus, Alertmanager) as a separate Docker Compose file with its own systemd unit. Also installs node_exporter natively for host-level metrics.
4. **PDS stats** — A cron-like systemd timer that runs a script to collect PDS-specific metrics and write them to a log file, which Promtail picks up into Loki.

All secrets are loaded from the encrypted `vars/secrets.yml` vault file.

### Templates

| Template                              | Deploys to                               | Purpose                                                                          |
| ------------------------------------- | ---------------------------------------- | -------------------------------------------------------------------------------- |
| `pds.env.j2`                          | `<pds_data_dir>/pds.env`                           | PDS environment config — hostname, secrets, feature flags                        |
| `docker-compose.pds.yml.j2`           | `<pds_data_dir>/compose.yaml`                      | PDS, Caddy (reverse proxy + TLS), and Watchtower (auto-updates all containers)   |
| `docker-compose.observability.yml.j2` | `<pds_data_dir>/docker-compose.observability.yml`  | Loki, Promtail, Prometheus, Alertmanager, and Grafana                            |
| `Caddyfile.j2`                        | `<pds_data_dir>/caddy/etc/caddy/Caddyfile`         | Reverse proxy rules — PDS with on-demand TLS for handle subdomains, plus Grafana |
| `grafana-datasources.yaml.j2`         | `<pds_data_dir>/grafana/provisioning/datasources/` | Auto-provisions Loki and Prometheus as Grafana data sources                      |
| `loki-config.yaml.j2`                 | `<pds_data_dir>/loki-config.yaml`                  | Loki storage and retention settings                                              |
| `promtail-config.yaml.j2`             | `<pds_data_dir>/promtail-config.yaml`              | Promtail scrape configs — Docker container logs and the PDS stats log            |
| `prometheus-config.yaml.j2`           | `<pds_data_dir>/prometheus-config.yaml`            | Prometheus scrape targets — node_exporter and PDS stats endpoint                 |
| `prometheus-alerts.yaml.j2`           | `<pds_data_dir>/prometheus-alerts.yaml`            | Alert rules for disk usage, high memory, PDS health, etc.                        |
| `alertmanager-config.yaml.j2`         | `<pds_data_dir>/alertmanager-config.yaml`          | Alert routing — sends notifications to a Discord webhook                         |
| `pds-stats.sh.j2`                     | `<pds_data_dir>/pds-stats.sh`                      | Shell script that queries PDS metrics and writes JSON to a log file              |

### Static files

| File                    | Deploys to                                  | Purpose                                                                         |
| ----------------------- | ------------------------------------------- | ------------------------------------------------------------------------------- |
| `node-exporter.service` | `/etc/systemd/system/node-exporter.service` | Systemd unit for the Prometheus node_exporter binary                            |
| `pds-stats.timer`       | `/etc/systemd/system/pds-stats.timer`       | Timer that triggers `pds-stats.service` on a schedule                           |
| `grafana/`              | `<pds_data_dir>/grafana/`                   | Custom dashboards (PDS overview, system metrics), CSS, and index.html overrides |

The following systemd units are deployed as **templates** (in `templates/`) since they reference `pds_data_dir`:

| Template                      | Deploys to                                  | Purpose                                                                         |
| ----------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------- |
| `pds.service.j2`              | `/etc/systemd/system/pds.service`           | Systemd unit that runs `docker compose up/down` for the PDS stack               |
| `observability.service.j2`    | `/etc/systemd/system/observability.service` | Systemd unit for the observability stack, ordered after `pds.service`           |
| `pds-stats.service.j2`        | `/etc/systemd/system/pds-stats.service`     | Oneshot service that runs the PDS stats collection script                       |

### Variables

#### `vars/main.yml`

Non-secret configuration. Gitignored — use `vars/main.yml.example` as a template.

#### `vars/secrets.yml`

Ansible Vault-encrypted file containing sensitive values. Gitignored — use `vars/secrets.yml.example` as a template. See [Editing secrets](#editing-secrets) above.

## Extras

### S3-compatible blob storage

By default, blobs are stored on disk at `<pds_data_dir>/blocks`. To use an S3-compatible provider (AWS S3, Cloudflare R2, Tigris, MinIO, etc.), set `pds_blobstore_type` to `s3` in `vars/main.yml` and uncomment the S3 variables:

```yaml
pds_blobstore_type: s3
pds_blobstore_s3_bucket: my-pds-blobs
pds_blobstore_s3_region: us-east-1
pds_blobstore_s3_endpoint: https://s3.example.com # omit for AWS S3
pds_blobstore_s3_force_path_style: true # needed for some S3-compatible providers
pds_blobstore_s3_access_key: "{{ vault_pds_blobstore_s3_access_key }}"
pds_blobstore_s3_secret_key: "{{ vault_pds_blobstore_s3_secret_key }}"
```

Then uncomment and fill in the S3 credentials in `vars/secrets.yml`:

```yaml
vault_pds_blobstore_s3_access_key: YOUR_ACCESS_KEY
vault_pds_blobstore_s3_secret_key: YOUR_SECRET_KEY
```
