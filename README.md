# Monitoring Stack

This project contains the configuration files for an infrastructure monitoring system where a monitoring server collects metrics and logs from an application server. Most of the repository is dedicated to the monitoring server setup; application server-specific configuration is kept separately under `app-server`.

## Architecture Overview

The system is split into two server roles:

- Application server: runs agents/exporters that expose monitoring data.
- Monitoring server: runs the services that collect, store, query, and visualize that data.

Main data flow:

```text
Application server
  node_exporter  :9100  -> Prometheus
  cAdvisor       :8088  -> Prometheus
  Promtail              -> Loki :3100

Monitoring server
  Prometheus -> Grafana
  Loki       -> Grafana
```

## Components

### Application Server

The application server is expected to run:

- `node_exporter`: exposes operating system metrics on port `9100`.
- `cAdvisor`: exposes container metrics on port `8088`.
- `promtail`: reads server logs and pushes them to Loki on the monitoring server.

This repository only stores the `promtail` configuration at `app-server/promtail-config.yml`, because `node_exporter` and `cAdvisor` do not require project-specific configuration files here.

### Monitoring Server

The monitoring server is deployed with `docker-compose.yml` and includes:

- `prometheus`: scrapes metrics from the application server and stores time-series data.
- `loki`: receives and stores logs from Promtail.
- `grafana`: provides dashboards, provisions Prometheus/Loki datasources, and supports alert delivery through SMTP.

Alerting note: this repository does not include a standalone `alertmanager` service and does not include Grafana alert rule/contact point provisioning files. The existing alert-related configuration is Grafana SMTP configuration, which allows Grafana to send email notifications when alert rules are created through the UI or added through provisioning later.

## Directory Structure

```text
.
|-- app-server/
|   `-- promtail-config.yml
|-- grafana/
|   `-- provisioning/
|       |-- dashboards/
|       |   `-- dashboards.yaml
|       |-- dashboards_json/
|       |   `-- 1860_rev45.json
|       `-- datasources/
|           `-- datasources.yaml
|-- loki/
|   `-- loki-config.yaml
|-- prometheus/
|   `-- prometheus.yml
|-- docker-compose.yml
`-- README.md
```

## Current Configuration

### Prometheus

File: `prometheus/prometheus.yml`

- `scrape_interval`: `15s`
- Scrape targets:
  - `10.140.0.3:9100` for `node_exporter`
  - `10.140.0.3:8088` for `cAdvisor`

If the application server address changes, update the IP address or hostname in this file.

### Loki

File: `loki/loki-config.yaml`

- Loki listens on port `3100`.
- Filesystem storage is used under `/var/lib/loki`.
- Single-node setup with `replication_factor: 1`.
- Analytics reporting is disabled.

### Promtail

File: `app-server/promtail-config.yml`

Promtail pushes logs to Loki at:

```text
http://34.80.22.72:3100/loki/api/v1/push
```

Configured log sources:

- `/var/log/syslog` with labels `job=syslog`, `host=vm2`
- `/var/log/auth.log` with labels `job=auth`, `host=vm2`
- `/var/lib/docker/containers/*/*-json.log` with labels `job=docker`, `host=vm2`

Configured pipeline stages:

- Drop syslog lines matching `systemd.*Starting.*`
- Drop syslog lines containing `CRON`
- Parse SSH messages from `auth.log`
- Parse Docker JSON logs with the `docker` stage

### Grafana

Grafana is provisioned from the `grafana/provisioning` directory.

Datasources:

- `Metric - Prometheus`: default datasource, points to `http://prometheus:9090`
- `Loki`: points to `http://loki:3100`

Dashboards:

- Provider: `Monitoring dashboard`
- Grafana folder: `Infrastructure`
- Dashboard JSON: `Node Exporter Full`, imported from dashboard ID `1860`

Grafana is also configured with SMTP in `docker-compose.yml` for email delivery. For production deployments, move SMTP credentials to a `.env` file or a secret manager.

## Requirements

- Docker
- Docker Compose
- The monitoring server must be able to reach the application server on:
  - `9100` for node_exporter
  - `8088` for cAdvisor
- The application server must be able to reach Loki on the monitoring server through port `3100`

## Run The Monitoring Server

From the repository root, run:

```bash
docker compose up -d
```

The services are exposed at:

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000`
- Loki: `http://localhost:3100`

Check container status:

```bash
docker compose ps
```

View logs when debugging:

```bash
docker compose logs -f prometheus
docker compose logs -f grafana
docker compose logs -f loki
```

## Configure The Application Server

On the application server, make sure that:

1. `node_exporter` is running and exposing port `9100`.
2. `cAdvisor` is running and exposing port `8088`.
3. `promtail` uses the configuration from `app-server/promtail-config.yml`.
4. The Loki address in Promtail `clients.url` points to the correct monitoring server.
5. Firewall or security group rules allow communication between the application server and the monitoring server on the required ports.

If the application server hostname or IP address changes, update:

- `prometheus/prometheus.yml`
- `app-server/promtail-config.yml` if the Loki address changes

## Data Storage

Docker Compose creates the following volumes:

- `prometheus_data`: stores Prometheus data.
- `grafana_data`: stores Grafana data.

Loki currently mounts data to the local directory `./loki-data:/var/lib/loki`. The `loki_data` volume is declared in `docker-compose.yml` but is not attached to the Loki service.

Prometheus retention is currently limited by:

- Time: `5d`
- Size: `3GB`

## Post-Deployment Checks

1. Open Prometheus at `http://localhost:9090/targets` and confirm that the `node_exporter` and `cadvisor` targets are `UP`.
2. Open Grafana at `http://localhost:3000` and verify the Prometheus/Loki datasources.
3. Open the `Infrastructure` folder in Grafana and view the `Node Exporter Full` dashboard.
4. In Grafana Explore, select Loki and run:

```logql
{host="vm2"}
```

## Security And Operations Notes

- Do not commit SMTP passwords directly in `docker-compose.yml`; use environment variables, a `.env` file, Docker secrets, or a secret manager.
- Expose only the required ports to the network.
- Change the default Grafana admin password after initialization.
- Add Grafana alert rule/contact point provisioning if alerts should be managed as code instead of being created manually in the Grafana UI.
