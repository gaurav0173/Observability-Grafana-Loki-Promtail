# Comprehensive Documentation: Setting Up Loki, Promtail, and Grafana with Docker Compose

## Introduction
Loki is a horizontally scalable, highly available, multi-tenant log aggregation system inspired by Prometheus, designed to store and query logs efficiently. Promtail is the agent responsible for collecting logs from various sources and sending them to Loki. Grafana provides a powerful, customizable dashboard for visualizing these logs. Using Docker Compose, we can orchestrate these services easily, ensuring they work together seamlessly.

This guide assumes you have Docker and Docker Compose installed on your system. The current time is 11:49 PM PDT on Monday, March 24, 2025, and all configurations are tailored for this setup.

## Creating an EC2 or Azure VM

### AWS EC2
1. Log in to your AWS Management Console.
2. Navigate to the **EC2 Dashboard** and click **Launch Instance**.
3. Choose an **Ubuntu** AMI (Amazon Machine Image), such as Ubuntu 22.04.
4. Select an instance type (t2.micro for testing or a larger instance for production).
5. Configure instance details and set up security groups to allow SSH (port 22), Grafana (port 3000), and Loki (port 3100).
6. Launch the instance and download the key pair for SSH access.

### Azure VM
1. Log in to the **Azure Portal**.
2. Navigate to **Virtual Machines** and click **Create**.
3. Select **Ubuntu 22.04 LTS** as the OS.
4. Choose an appropriate size (e.g., **Standard_B1s** for testing).
5. Configure **networking** to allow SSH (port 22) and other necessary ports.
6. Click **Review + Create** and deploy the VM.
7. Edit inbound rules in **Networking > Inbound port rules**:
   - Allow **SSH (22)** from your IP.
   - Allow **Grafana (3000)** and **Loki (3100)** from trusted sources.


### Connecting to the Instance
Once the VM is running, connect via SSH:

For AWS EC2:
```bash
ssh -i your-key.pem ubuntu@<your-ec2-ip>
```

For Azure VM:
```bash
ssh azureuser@<your-azure-vm-ip>
```

## Directory Structure and File Creation
First, create a directory to organize your configuration files:

### Create a directory named monitoring:
```bash
mkdir monitoring
cd monitoring
```

Inside the `monitoring` directory, create the following files:
- `docker-compose.yml`: Defines the Docker Compose configuration for Loki, Promtail, and Grafana.
- `loki-config.yml`: Configuration file for Loki, including WAL, schema, and storage settings.
- `promtail-config.yml`: Configuration file for Promtail, defining log scraping and Loki endpoint.

```

(Additional configurations and instructions remain unchanged)

This setup ensures a robust log monitoring system with a colorful, visual dashboard in Grafana, addressing previous issues and providing a complete solution.


```

Inside the `monitoring` directory, create the following files:
- `docker-compose.yml`: Defines the Docker Compose configuration for Loki, Promtail, and Grafana.
- `loki-config.yml`: Configuration file for Loki, including WAL, schema, and storage settings.
- `promtail-config.yml`: Configuration file for Promtail, defining log scraping and Loki endpoint.


### Create and edit the configuration files using nano:
```bash
nano docker-compose.yml
```
Copy and paste the appropriate `docker-compose.yml` content, then save and exit (Ctrl+X, Y, Enter).

```bash
nano loki-config.yml
```
Copy and paste the appropriate `loki-config.yml` content, then save and exit.

```bash
nano promtail-config.yml
```
Copy and paste the appropriate `promtail-config.yml` content, then save and exit.


## Configuring loki-config.yml
Loki requires a configuration file to define its behavior. Here’s the configuration:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  wal:
    enabled: true
    dir: /tmp/loki/wal

schema_config:
  configs:
    - from: 2022-06-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/index
    cache_location: /tmp/loki/cache
  filesystem:
    directory: /tmp/loki/chunks

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  allow_structured_metadata: false

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s

compactor:
  working_directory: /tmp/loki/compactor
```

## Configuring promtail-config.yml

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
```

## Configuring docker-compose.yml

```yaml
version: '3.8'

services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - ./loki-data:/tmp/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    restart: unless-stopped

volumes:
  grafana-storage:
```

## Setting Up the Stack

Create the `loki-data` directory for persistence:

```bash
mkdir -p ./loki-data
```

Start the Docker Compose stack:

```bash
docker compose up -d
```

Verify the containers are running:

```bash
docker ps
```

## Accessing and Configuring Grafana

1. Open your browser and navigate to `http://<your-server-ip>:3000`.
2. Log in with username `admin` and password `admin`.
3. Add Loki as a data source:
   - Go to Configuration (gear icon) > Data Sources.
   - Click Add data source, select Loki.
   - Set Name to `Loki`, URL to `http://loki:3100`, and click `Save & Test`.

## Creating a Grafana Dashboard for Log Visualization

1. Go to Dashboards > Create new dashboard.
2. Click `Add new panel`.
3. Select `Loki` as the data source.
4. Enter a LogQL query, e.g., `{job="varlogs"}`.
5. Set the visualization:
   - Use Logs panel for raw logs.
   - Use Time Series for graphs.
6. Customize colors:
   - Go to Field tab > Standard options > Color scheme.
   - Add thresholds (e.g., green for <10 logs/min, red for >50).
7. Click `Apply` and save the dashboard.

## Generating Test Logs

```bash
echo "Test log entry $(date)" >> /var/log/test.log
```

Check the dashboard to ensure logs appear.

## Troubleshooting

- If Loki fails to start, check logs with:

  ```bash
  docker logs loki
  ```

- Ensure Promtail is sending logs by checking:

  ```bash
  docker logs promtail
  ```

- If no logs appear in Grafana, verify the data source connection and LogQL query syntax in `Explore`.

## Summary Table of Configuration Files

| File Name          | Purpose                                   | Key Configuration Points          |
|--------------------|-------------------------------------------|----------------------------------|
| `loki-config.yml` | Loki configuration                      | WAL dir, storage paths, schema v12 |
| `promtail-config.yml` | Promtail log collection configuration | Scrapes `/var/log/*log`, sends to Loki |
| `docker-compose.yml` | Docker Compose orchestration           | Maps volumes, exposes ports 3100 and 3000 |

This setup ensures a robust log monitoring system with a complete solution for visualization in Grafana.

