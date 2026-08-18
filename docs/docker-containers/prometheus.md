# Prometheus

[Uptime Kuma](uptime-kuma.md) answers "is it up?". Prometheus answers "why was it slow last Tuesday at 9pm?" — it records numbers over time, so you can look backwards instead of guessing.

Prometheus is the collection half. [Grafana](grafana.md) is the part that draws the graphs. Set this up first; it is useless alone but Grafana is useless without it.

[Prometheus](https://prometheus.io/) is a time-series database and monitoring system that scrapes metrics from HTTP endpoints on a schedule.

1. **Pull-based** — it fetches from targets, rather than services pushing to it. Nothing to configure at the sending end beyond exposing a page.
2. **PromQL**, a query language built for time series, including rates, aggregation and prediction
3. **Exporters** for essentially everything — hosts, containers, Proxmox, network gear, databases
4. **Alertmanager** for rules with grouping, silencing and routing
5. **Efficient storage** — a homelab's metrics for a year fit in a few gigabytes
6. **Service discovery**, so targets can appear and disappear without editing config

## The pieces

Prometheus alone collects nothing about your machines. You need exporters — small programs that expose metrics on an HTTP endpoint:

| Exporter | Provides | Port |
|---|---|---|
| **node-exporter** | CPU, RAM, disk, network, temperature for a Linux host | 9100 |
| **cAdvisor** | Per-container CPU, memory, network | 8080 |
| **Prometheus** | Its own metrics | 9090 |

Add more later — the Proxmox exporter, an SNMP exporter for switches, or whatever your services expose natively.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - A host that is not already using ports 9090, 9100 or 8080
    - Somewhere to put a few GB of metrics

## 1. Create the stack

```bash
sudo mkdir -p /opt/docker/monitoring && cd /opt/docker/monitoring
sudo nano docker-compose.yml
```

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    user: "65534:65534"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=90d'
      - '--web.enable-lifecycle'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports:
      - 9090:9090

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    command:
      - '--path.rootfs=/host'
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    pid: host
    volumes:
      - /:/host:ro,rslave
    ports:
      - 9100:9100

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - 8082:8080

volumes:
  prometheus_data:
```

Things worth understanding:

1. **`user: "65534:65534"`** — runs as `nobody`. Prometheus does not need root, and the data volume is owned accordingly.
2. **`--storage.tsdb.retention.time=90d`** — how far back you can look. 90 days of homelab metrics is a few GB. The default is 15 days, which is not enough to spot a monthly pattern.
3. **`--web.enable-lifecycle`** — allows config reload over HTTP instead of restarting the container.
4. **`pid: host`** on node-exporter — it needs the host's process namespace to report accurately.
5. **cAdvisor on host port 8082** — it uses 8080 internally, which collides with almost everything. Remap it.
6. **The `$$` in the exclude regex** — Compose interprets `$`, so it must be escaped. A single `$` here produces a confusing startup failure.

## 2. Write the config

```bash
sudo nano prometheus.yml
```

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
        labels:
          instance: 'proxmox2'

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
        labels:
          instance: 'proxmox2'
```

`scrape_interval: 15s` is a sensible homelab default. Going to 5s quadruples your storage for detail you will not use.

Note cAdvisor is `cadvisor:8080` here — the *container* port, since Prometheus reaches it over the Docker network.

```bash
docker compose up -d
docker compose logs -f prometheus
```

## 3. Check the targets

Browse to `http://<host-ip>:9090`, then **Status → Targets**.

All three should be **UP**. A red target tells you exactly what failed — usually a name that does not resolve, or a port that is the host-side mapping rather than the container's.

Try a query in the **Graph** tab:

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

That is CPU usage as a percentage. If it returns a number, the whole pipeline works.

<!-- screenshot: Prometheus targets page showing all endpoints UP -->

## 4. Add more hosts

Monitoring one machine is not the point. Deploy node-exporter to your other hosts:

```yaml
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    command:
      - '--path.rootfs=/host'
    pid: host
    volumes:
      - /:/host:ro,rslave
    ports:
      - 9100:9100
```

Then add them to `prometheus.yml`:

```yaml
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
        labels:
          instance: 'proxmox2'
      - targets: ['192.168.1.10:9100']
        labels:
          instance: 'proxmox1'
      - targets: ['192.168.1.50:9100']
        labels:
          instance: 'unraid'
```

The `instance` label is what you will group by in Grafana, so make it a name you recognise rather than an IP.

Reload without restarting:

```bash
curl -X POST http://localhost:9090/-/reload
```

!!! warning "node-exporter has no authentication"

    Port 9100 exposes detailed information about the host — filesystems, network interfaces, running processes count, uptime. It is not catastrophic, but it is reconnaissance. Firewall it to the Prometheus host:

    ```bash
    sudo ufw allow from 192.168.1.30 to any port 9100 proto tcp
    ```

## 5. Monitor Proxmox itself

The [Proxmox exporter](https://github.com/prometheus-pve/prometheus-pve-exporter) reports on the hypervisor: VM states, storage usage, cluster health.

```yaml
  pve-exporter:
    image: prompve/prometheus-pve-exporter:latest
    container_name: pve-exporter
    restart: unless-stopped
    environment:
      - PVE_USER=prometheus@pve
      - PVE_TOKEN_NAME=monitoring
      - PVE_TOKEN_VALUE=${PVE_TOKEN}
      - PVE_VERIFY_SSL=false
    ports:
      - 9221:9221
```

Create a read-only API token in Proxmox first: **Datacenter → Permissions → API Tokens**, user `prometheus@pve`, role `PVEAuditor`. Audit-only means a leaked token reads metrics and nothing else.

```yaml
  - job_name: 'pve'
    static_configs:
      - targets: ['192.168.1.10']
    metrics_path: /pve
    params:
      module: [default]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: pve-exporter:9221
```

The `relabel_configs` block looks strange but is standard for exporters that proxy to a target: Prometheus scrapes the exporter, telling it which Proxmox node to query.

## 6. Alerting rules, optionally

Prometheus can alert on trends, which is the thing Uptime Kuma cannot do — "disk will be full in four hours" rather than "disk is full".

```bash
sudo nano alerts.yml
```

```yaml
groups:
  - name: homelab
    rules:
      - alert: DiskWillFillIn24Hours
        expr: predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"}[6h], 24*3600) < 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} {{ $labels.mountpoint }} will fill within 24h"

      - alert: HostHighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} CPU above 90% for 15 minutes"

      - alert: HostOutOfMemory
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} has less than 10% memory available"
```

Reference it in `prometheus.yml`:

```yaml
rule_files:
  - /etc/prometheus/alerts.yml
```

And mount it: `- ./alerts.yml:/etc/prometheus/alerts.yml:ro`.

Delivering those alerts needs Alertmanager, or you can let Grafana handle alerting instead — which for a homelab is usually the simpler path, since you are already going to be in Grafana.

## Updating

```bash
cd /opt/docker/monitoring
docker compose pull
docker compose up -d
```

Prometheus is careful about storage compatibility, so upgrades are generally uneventful.

## Backup

Metrics are, arguably, disposable — losing them costs you history, not function. The config is not.

```bash
sudo tar czf /mnt/user/backups/prometheus-config-$(date +%F).tar.gz \
  -C /opt/docker/monitoring prometheus.yml alerts.yml docker-compose.yml
```

If you do want the data, snapshot it properly rather than copying files under a running server:

```bash
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot
```

That needs `--web.enable-admin-api` added to the command list.

## Troubleshooting

**Target shows "connection refused".** Wrong port — you likely used the host-side mapping. Inside the Docker network, cAdvisor is `8080`, not `8082`.

**Target shows "context deadline exceeded".** The exporter is too slow, or the scrape interval is shorter than the scrape takes. Raise `scrape_timeout`.

**node-exporter reports the container's filesystem, not the host's.** The `--path.rootfs=/host` flag or the `/:/host:ro,rslave` mount is missing.

**cAdvisor uses a lot of CPU.** Known behaviour on hosts with many containers. Add `--housekeeping_interval=30s` to reduce it.

**"invalid reference format" or a `$` error on startup.** The escaping in the filesystem exclude regex. It needs `$$` in a Compose file.

**Prometheus will not start: "opening storage failed ... permission denied".** The `user:` directive and the volume ownership disagree. `docker run --rm -v prometheus_data:/data alpine chown -R 65534:65534 /data`.

**Disk filling up.** Lower the retention, or check you have not set a very short scrape interval across many targets.

## Where this sits in my lab

Prometheus runs on `proxmox2` with node-exporter on all three hosts, cAdvisor on the two Docker hosts, and the PVE exporter against both hypervisors.

90 days of retention, 15-second scrapes. It answers the questions Uptime Kuma cannot: whether the array is filling faster this month than last, which container started eating RAM after an update, and whether that evening slowdown is CPU, disk or the network. [Grafana](grafana.md) is where any of that becomes visible.
