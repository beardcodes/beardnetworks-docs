# Grafana

[Prometheus](prometheus.md) collects the numbers. Grafana is where they become something you can look at — dashboards, alerts, and the graph you screenshot when someone asks why the internet was slow last night.

Set this up after Prometheus, since a Grafana with no data source is an empty room.

[Grafana](https://grafana.com/) is an open-source visualisation and alerting platform.

1. **Dashboards** — panels, rows, variables, drill-downs, all editable in the browser
2. **Many data sources** — Prometheus, Loki, InfluxDB, SQL databases, even a CSV
3. **Thousands of community dashboards** you can import by ID rather than building from scratch
4. **Unified alerting** with its own rule engine and notification routing
5. **Provisioning as code**, so data sources and dashboards can live in your Git repo
6. **Free and open source** — the OSS build has everything a homelab needs

## 1. The container

Add to the same monitoring stack as Prometheus:

```yaml
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    restart: unless-stopped
    user: "472"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://192.168.1.30:3000
      - GF_ANALYTICS_REPORTING_ENABLED=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - 3000:3000
    depends_on:
      - prometheus

volumes:
  grafana_data:
```

```bash
openssl rand -base64 24
echo "GRAFANA_PASSWORD=<that string>" >> .env
chmod 600 .env
```

Notes:

1. **`user: "472"`** is the `grafana` user inside the image. The data volume must be owned by it.
2. **`GF_USERS_ALLOW_SIGN_UP=false`** — otherwise anyone reaching the page can create an account.
3. **`GF_SERVER_ROOT_URL`** — used for links in alert notifications. Set it to how you actually reach Grafana, or every alert links somewhere useless.
4. **Port 3000** collides with a lot of things — [Karakeep](karakeep.md) among them. Remap the host side if they share a machine.

## 2. Provision the data source

You *can* click through the UI. Better to define it in a file, so a rebuilt container comes back configured.

```bash
sudo mkdir -p grafana/provisioning/datasources
sudo nano grafana/provisioning/datasources/prometheus.yml
```

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

`access: proxy` means Grafana's backend queries Prometheus, so your browser never needs to reach port 9090 — which lets you keep Prometheus entirely off the network your workstation sits on.

```bash
docker compose up -d grafana
```

## 3. Log in

`http://<host-ip>:3000`, with `admin` and the password from your `.env`.

Confirm the data source under **Connections → Data sources → Prometheus → Test**. Green means Grafana can reach it.

## 4. Import dashboards instead of building them

This is the part people miss. Do not build a node dashboard by hand — thousands already exist.

**Dashboards → New → Import**, then enter an ID:

| ID | Dashboard | Needs |
|---|---|---|
| **1860** | Node Exporter Full | node-exporter |
| **193** | Docker and system monitoring | cAdvisor |
| **19792** | Proxmox via PVE exporter | pve-exporter |
| **13946** | Docker cAdvisor compute resources | cAdvisor |

Select Prometheus as the data source when prompted.

**1860 is the one to start with.** It is a comprehensive host dashboard — CPU, memory, disk I/O, network, filesystem, temperatures — and it works immediately with the node-exporter setup from the Prometheus guide.

<!-- screenshot: Grafana Node Exporter Full dashboard with CPU and memory panels -->

Browse the rest at [grafana.com/dashboards](https://grafana.com/grafana/dashboards/).

## 5. Understand enough PromQL to be dangerous

Imported dashboards cover most needs, but a few queries are worth knowing when you want a panel of your own:

```promql
# CPU usage percentage, per host
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory used percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Root filesystem free, in GB
node_filesystem_avail_bytes{mountpoint="/"} / 1024 / 1024 / 1024

# Container memory, top 10
topk(10, container_memory_usage_bytes{name!=""})

# Network receive rate, per interface
rate(node_network_receive_bytes_total[5m]) * 8

# Days until this filesystem fills, based on the last 6 hours
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 86400) < 0
```

Two rules cover most of it: use `rate()` on anything whose name ends in `_total` (they are counters, and the raw value only goes up), and use `avg by (instance)` to collapse per-CPU or per-core series into one line per host.

## 6. Add a variable, so one dashboard covers every host

**Dashboard settings → Variables → New variable**:

| Field | Value |
|---|---|
| Name | `instance` |
| Type | Query |
| Data source | Prometheus |
| Query | `label_values(node_uname_info, instance)` |
| Multi-value | On |

Then use `{instance=~"$instance"}` in your panel queries, and you get a dropdown at the top of the dashboard to switch between hosts. Most imported dashboards already do this.

## 7. Alerting

Grafana's built-in alerting is usually easier than running Alertmanager separately, and you are already here.

**Alerting → Alert rules → New alert rule**:

1. **Query** — e.g. `node_filesystem_avail_bytes{mountpoint="/"} / 1024^3`
2. **Condition** — `IS BELOW 10`
3. **Evaluation** — every 5m, pending period 10m
4. **Labels and notifications** — pick a contact point

Then **Alerting → Contact points → Add contact point**. ntfy works through the webhook type; Discord and Telegram have native entries.

!!! note "Grafana alerts and Uptime Kuma do different jobs"

    Keep both, and split them by question:

    - **[Uptime Kuma](uptime-kuma.md)**: is it up? Binary, immediate, low-noise.
    - **Grafana**: is something trending badly? Disk filling, memory creeping, CPU sustained, temperature climbing.

    Duplicating "service down" alerts across both just trains you to ignore your phone.

## 8. Put it behind Traefik

```yaml
    environment:
      - GF_SERVER_ROOT_URL=https://grafana.example.com
    networks:
      - default
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.example.com`)"
      - "traefik.http.routers.grafana.entrypoints=https"
      - "traefik.http.routers.grafana.tls.certresolver=letsencrypt"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"
      - "traefik.docker.network=traefik-net"
```

Update `GF_SERVER_ROOT_URL` at the same time, or alert links keep pointing at the old address.

Grafana handles its own authentication, so it does not need an auth middleware in front — but it should still stay on the LAN or behind [wg-easy](wg-easy.md) unless you have a specific reason to publish it.

## Updating

```bash
cd /opt/docker/monitoring
docker compose pull grafana
docker compose up -d grafana
```

Grafana migrates its database on start. Major versions have occasionally changed panel or alerting formats — back up first and skim the release notes.

## Backup

```bash
docker compose stop grafana
docker run --rm \
  -v monitoring_grafana_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/grafana-$(date +%F).tar.gz -C /data .
docker compose start grafana
```

The volume holds `grafana.db` — dashboards, users, alert rules, API keys.

Better still, provision dashboards from files as well as data sources. A dashboard defined in JSON in your Git repo survives losing the volume entirely:

```yaml
apiVersion: 1
providers:
  - name: 'homelab'
    folder: 'Homelab'
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

Export any dashboard as JSON from its share menu and drop it in that folder.

## Troubleshooting

**"HTTP Error Bad Gateway" testing the data source.** Grafana cannot reach Prometheus. Use `http://prometheus:9090` — container name, container port, and both must be on the same Docker network.

**Dashboard imports but every panel says "No data".** Either the data source was not selected during import, or the dashboard expects metric names your exporters do not produce. Check one panel's query against the Prometheus UI directly.

**Grafana will not start: "permission denied" on the database.** Volume ownership. `docker run --rm -v monitoring_grafana_data:/data alpine chown -R 472:472 /data`.

**Locked out — forgotten admin password.**

```bash
docker exec -it grafana grafana-cli admin reset-admin-password newpassword
```

**Alert notification links point at localhost.** `GF_SERVER_ROOT_URL` is wrong.

**Panels are slow, or time out on long ranges.** The query is asking for too many points. Increase the `rate()` window, or set a minimum step interval on the panel.

**Everything shows an hour off.** Timezone. Set it per-dashboard, or `TZ` on the container.

**"Too many outstanding requests" from Prometheus.** A dashboard is firing dozens of queries at once, often because a variable is set to "All" across many hosts. Narrow it.

## Where this sits in my lab

Grafana runs on `proxmox2` next to Prometheus, reading from it over the internal Docker network so Prometheus never needs to be reachable from a browser.

Three dashboards get used: Node Exporter Full for the hosts, a cAdvisor one for containers, and the Proxmox dashboard for VM and storage state. Everything else I imported at some point and never opened again — which is the honest outcome for most Grafana installs, and the reason importing beats building.

Alerts here are strictly about trends. "Is it down" belongs to [Uptime Kuma](uptime-kuma.md); "the array will be full in a fortnight" belongs here.
