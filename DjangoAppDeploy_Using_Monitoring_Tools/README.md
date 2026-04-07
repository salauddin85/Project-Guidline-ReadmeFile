# NewsPortal - Complete Deployment & Monitoring Guide

**Author:** Salauddin
**Role:** Software Developer

A comprehensive guide to deploy Django NewsPortal on EC2/VPS with full monitoring stack (Prometheus, Grafana, Loki, Promtail, AlertManager).

## 📋 Table of Contents
- Architecture Overview
- Prerequisites
- Part 1: Initial Server Setup
- Part 2: Django Application Deployment
- Part 3: Nginx Configuration
- Part 4: Prometheus Installation
- Part 5: Node Exporter Installation
- Part 6: Grafana Installation
- Part 7: Loki Installation
- Part 8: Promtail Installation
- Part 9: AlertManager Installation
- Part 10: Grafana Configuration
- Part 11: Viewing Logs in Loki
- Part 12: Service Management
- Part 13: Troubleshooting

---

## Architecture Overview

================================================================================
                            ARCHITECTURE OVERVIEW
================================================================================

+==============================================================================+
|                           VPS / EC2 INSTANCE                                  |
+==============================================================================+
|                                                                              |
|   +-------------+     +-------------+     +-------------+                   |
|   |   Nginx     |---->|   Django    |---->|  Postgres   |                   |
|   |   Port 80   |     |  Port 8000  |     | (optional)  |                   |
|   +-------------+     +-------------+     +-------------+                   |
|          |                   |                                              |
|          |                   |                                              |
|          v                   v                                              |
|   +-------------+     +-------------+     +-------------+                   |
|   |  Promtail   |---->|    Loki     |     | Prometheus  |                   |
|   |   (Logs)    |     |  Port 3100  |     |  Port 9090  |                   |
|   +-------------+     +-------------+     +-------------+                   |
|          |                   |                   |                          |
|          |                   |                   |                          |
|          |                   v                   v                          |
|          |            +-------------+     +-------------+                   |
|          |            |   Grafana   |<----|    Node     |                   |
|          |            |  Port 3000  |     |  Exporter   |                   |
|          |            +-------------+     |  Port 9100  |                   |
|          |                   |            +-------------+                   |
|          |                   |                   |                          |
|          |                   |                   v                          |
|          |                   |            +-------------+                   |
|          |                   +----------->|AlertManager |                   |
|          |                                |  Port 9093  |                   |
|          |                                +-------------+                   |
|          |                                       |                          |
|          v                                       v                          |
|   +-------------+                           +-------------+                 |
|   |  LOG FILES  |                           |  SLACK      |                 |
|   |  /var/log/  |                           |  / EMAIL    |                 |
|   +-------------+                           +-------------+                 |
|                                                                              |
+==============================================================================+

================================================================================
                            COMPONENT DETAILS
================================================================================

+----------------+----------+-----------------------------------------------+
|   Component    |   Port   |              Description                       |
+----------------+----------+-----------------------------------------------+
| Nginx          | 80       | Reverse proxy for Django application          |
| Django         | 8000     | NewsPortal web application                    |
| PostgreSQL     | 5432     | Database (optional)                           |
| Prometheus     | 9090     | Metrics collection and storage                |
| Node Exporter  | 9100     | Server metrics (CPU, RAM, Disk, Network)      |
| Loki           | 3100     | Log aggregation system                        |
| Promtail       | -        | Log shipping agent                            |
| Grafana        | 3000     | Visualization dashboard                       |
| AlertManager   | 9093     | Alert routing and notifications               |
+----------------+----------+-----------------------------------------------+
---

## Prerequisites

- VPS/EC2 with Ubuntu 22.04 or 24.04
- Minimum 2GB RAM, 20GB storage
- SSH access to your server
- Domain name (optional) or public IP
- Slack Webhook URL (optional, for alerts)

---

## Part 1: Initial Server Setup

Run this script as ubuntu user:

#!/bin/bash

# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y python3 python3-pip python3-venv python3-dev \
    git nginx curl wget unzip build-essential \
    libpq-dev libssl-dev libffi-dev ufw

# Configure firewall
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw allow 3000/tcp    # Grafana
sudo ufw enable

# Create log directories
sudo mkdir -p /var/log/newsportal
sudo chown ubuntu:ubuntu /var/log/newsportal

---

## Part 2: Django Application Deployment

# Upload your project from local machine
# On your local machine:
# zip -r newsportal.zip newsportal/
# scp -i your-key.pem newsportal.zip ubuntu@YOUR_EC2_IP:~/

# On VPS:
cd ~
unzip newsportal.zip
cd newsportal

# Create virtual environment
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# Create .env file
cat > .env <<'EOF'
SECRET_KEY=your-super-secret-key-here
DEBUG=False
ALLOWED_HOSTS=YOUR_EC2_IP,localhost
DATABASE_URL=sqlite:///db.sqlite3
EOF

# Setup database and static files
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser

# Create Gunicorn systemd service
sudo tee /etc/systemd/system/newsportal.service > /dev/null <<'EOF'
[Unit]
Description=NewsPortal Django App (Gunicorn)
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/newsportal
Environment="PATH=/home/ubuntu/newsportal/venv/bin"
EnvironmentFile=/home/ubuntu/newsportal/.env
ExecStart=/home/ubuntu/newsportal/venv/bin/gunicorn \
    --workers 3 \
    --bind 127.0.0.1:8000 \
    newsportal.wsgi:application
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start newsportal
sudo systemctl enable newsportal

---

## Part 3: Nginx Configuration

sudo tee /etc/nginx/sites-available/newsportal > /dev/null <<'EOF'
server {
    listen 80;
    server_name YOUR_EC2_IP;

    client_max_body_size 10M;

    location /static/ {
        alias /home/ubuntu/newsportal/staticfiles/;
        expires 30d;
    }

    location /media/ {
        alias /home/ubuntu/newsportal/media/;
        expires 7d;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /metrics {
        proxy_pass http://127.0.0.1:8000/metrics;
        proxy_set_header Host $host;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/newsportal /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx

---

## Part 4: Prometheus Installation

# Create prometheus user
sudo useradd --no-create-home --shell /bin/false prometheus

# Create directories
sudo mkdir -p /etc/prometheus /var/lib/prometheus

# Download and install
cd /tmp
wget -q https://github.com/prometheus/prometheus/releases/download/v2.51.0/prometheus-2.51.0.linux-amd64.tar.gz
tar xf prometheus-2.51.0.linux-amd64.tar.gz
cd prometheus-2.51.0.linux-amd64

sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# Configure Prometheus
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - "/etc/prometheus/alert_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'newsportal-server'

  - job_name: 'django_newsportal'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'newsportal-app'
EOF

# Create alert rules
sudo tee /etc/prometheus/alert_rules.yml > /dev/null <<'EOF'
groups:
  - name: server_health
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is DOWN"
          description: "{{ $labels.instance }} has been down for >1 minute."

      - alert: HighCPUUsage
        expr: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU usage is {{ printf \"%.1f\" $value }}%"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High RAM on {{ $labels.instance }}"
          description: "Memory usage is {{ printf \"%.1f\" $value }}%"
EOF

sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml /etc/prometheus/alert_rules.yml

# Create systemd service
sudo tee /etc/systemd/system/prometheus.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus Monitoring
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --storage.tsdb.retention.time=15d
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus

---

## Part 5: Node Exporter Installation

cd /tmp
wget -q https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xf node_exporter-1.7.0.linux-amd64.tar.gz
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

sudo useradd --no-create-home --shell /bin/false node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

---

## Part 6: Grafana Installation

# Add Grafana repository
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | \
    sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
    sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install -y grafana

# Configure Grafana
sudo tee -a /etc/grafana/grafana.ini > /dev/null <<'EOF'

[server]
http_port = 3000
domain = localhost

[security]
admin_user = admin
admin_password = admin123

[auth.anonymous]
enabled = false
EOF

sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

echo "✅ Grafana: http://YOUR_IP:3000 | Login: admin / admin123"

---

## Part 7: Loki Installation

IMPORTANT: The configuration below includes the required delete_request_store setting to avoid the "empty ring" error.

cd /tmp
wget -q https://github.com/grafana/loki/releases/download/v3.0.0/loki-linux-amd64.zip
unzip -q loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki

sudo mkdir -p /etc/loki /var/lib/loki/{chunks,rules,compactor}

sudo tee /etc/loki/loki-config.yml > /dev/null <<'EOF'
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: warn

common:
  instance_addr: 127.0.0.1
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 30d
  ingestion_rate_mb: 4
  ingestion_burst_size_mb: 6

ruler:
  alertmanager_url: http://localhost:9093

compactor:
  working_directory: /var/lib/loki/compactor
  retention_enabled: true
  delete_request_store: filesystem
EOF

sudo tee /etc/systemd/system/loki.service > /dev/null <<'EOF'
[Unit]
Description=Loki Log Aggregation
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start loki
sudo systemctl enable loki

---

## Part 8: Promtail Installation

cd /tmp
wget -q https://github.com/grafana/loki/releases/download/v3.0.0/promtail-linux-amd64.zip
unzip -q promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail

sudo mkdir -p /etc/promtail
sudo mkdir -p /home/ubuntu/newsportal/logs
sudo chown ubuntu:ubuntu /home/ubuntu/newsportal/logs

sudo tee /etc/promtail/promtail-config.yml > /dev/null <<'EOF'
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/promtail-positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          host: newsportal-server
          __path__: /var/log/syslog

  - job_name: nginx_access
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          log_type: access
          host: newsportal-server
          __path__: /var/log/nginx/access.log

  - job_name: nginx_error
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          log_type: error
          host: newsportal-server
          __path__: /var/log/nginx/error.log

  - job_name: django_app
    static_configs:
      - targets: [localhost]
        labels:
          job: django
          app: newsportal
          host: newsportal-server
          __path__: /home/ubuntu/newsportal/logs/*.log
EOF

sudo tee /etc/systemd/system/promtail.service > /dev/null <<'EOF'
[Unit]
Description=Promtail Log Shipper
After=network.target loki.service

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail-config.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start promtail
sudo systemctl enable promtail

---

## Part 9: AlertManager Installation

cd /tmp
wget -q https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xf alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64

sudo cp alertmanager amtool /usr/local/bin/
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo chown -R prometheus:prometheus /etc/alertmanager /var/lib/alertmanager

# Configure AlertManager (replace YOUR_SLACK_WEBHOOK_URL)
sudo tee /etc/alertmanager/alertmanager.yml > /dev/null <<'EOF'
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-receiver'

  routes:
    - match:
        severity: critical
      receiver: 'critical-receiver'
      repeat_interval: 1h

receivers:
  - name: 'default-receiver'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'
        channel: '#alerts'
        title: '⚠️ [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'critical-receiver'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'
        channel: '#critical-alerts'
        title: '🚨 CRITICAL: {{ .CommonLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}{{ end }}'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname']
EOF

sudo chown prometheus:prometheus /etc/alertmanager/alertmanager.yml

sudo tee /etc/systemd/system/alertmanager.service > /dev/null <<'EOF'
[Unit]
Description=AlertManager
After=network.target prometheus.service

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/alertmanager \
    --config.file=/etc/alertmanager/alertmanager.yml \
    --storage.path=/var/lib/alertmanager/
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start alertmanager
sudo systemctl enable alertmanager

---

## Part 10: Grafana Configuration

### Step 10.1: Login to Grafana

Open browser: http://YOUR_EC2_IP:3000
- Username: admin
- Password: admin123

### Step 10.2: Add Prometheus Data Source

1. Left sidebar → Connections → Data sources
2. Click Add data source → Select Prometheus
3. Configure:
   - Name: Prometheus
   - URL: http://localhost:9090
4. Click Save & Test → Should show "Successfully queried"

### Step 10.3: Add Loki Data Source

1. Add data source → Select Loki
2. Configure:
   - Name: Loki
   - URL: http://localhost:3100
3. Click Save & Test → Should show "Data source connected"

### Step 10.4: Import Dashboards

| Dashboard | Description | Import ID |
|-----------|-------------|-----------|
| Node Exporter Full | Server metrics (CPU, RAM, Disk, Network) | 1860 |
| Django Prometheus | Django application metrics | 17658 |
| Loki Logs Dashboard | Log viewing dashboard | 13639 |

Import Steps:
1. Left sidebar → Dashboards → Import
2. Enter Dashboard ID → Click Load
3. Select Prometheus/Loki as data source → Click Import

---

## Part 11: Viewing Logs in Loki

### Method 1: Using Grafana Explore

1. Left sidebar → Explore (compass icon)
2. Select Loki as data source
3. Enter LogQL queries:

# View all Django logs
{job="django"}

# View Nginx access logs
{job="nginx", log_type="access"}

# View Nginx error logs
{job="nginx", log_type="error"}

# View system logs
{job="syslog"}

# Filter ERROR level logs
{job="django"} |= "ERROR"

# Filter multiple keywords
{job="django"} |~ "(?i)error|fail|critical"

# Nginx 5xx errors
{job="nginx", log_type="access"} |~ "HTTP/1.1\" 5[0-9]{2}"

### Method 2: Using Grafana Dashboard

1. Go to Dashboards → Loki Logs Dashboard (ID: 13639)
2. Select labels from dropdown:
   - Job: django, nginx, or syslog
   - Host: newsportal-server
3. Click Show Logs

### Method 3: Using Command Line

# Query Loki API directly
curl -G 'http://localhost:3100/loki/api/v1/query' \
  --data-urlencode 'query={job="django"}' \
  --data-urlencode 'limit=10' \
  2>/dev/null | python3 -m json.tool

# Check available labels
curl -s 'http://localhost:3100/loki/api/v1/labels' | python3 -m json.tool

# Check job values
curl -s 'http://localhost:3100/loki/api/v1/label/job/values' | python3 -m json.tool

### Generate Test Logs

# Create test log entry
echo "INFO $(date '+%Y-%m-%d %H:%M:%S,%3N') Test application log" | tee -a /home/ubuntu/newsportal/logs/app.log

# Generate error log
echo "ERROR $(date '+%Y-%m-%d %H:%M:%S,%3N') Test error occurred" | tee -a /home/ubuntu/newsportal/logs/errors.log

# Generate nginx log
sudo bash -c 'echo "$(date) - Test request" >> /var/log/nginx/access.log'

# Generate system log
logger "Test log from newsportal server"

---

## Part 12: Service Management

### Check All Services Status

#!/bin/bash
echo "═══════════════════════════════════════════"
echo "  Monitoring Stack Status"
echo "═══════════════════════════════════════════"

services=("newsportal" "nginx" "prometheus" "node_exporter" "grafana-server" "loki" "promtail" "alertmanager")

for svc in "${services[@]}"; do
    status=$(systemctl is-active "$svc" 2>/dev/null)
    if [ "$status" = "active" ]; then
        echo "  ✅ $svc — running"
    else
        echo "  ❌ $svc — $status"
    fi
done

echo ""
echo "═══════════════════════════════════════════"
echo "  Health Checks"
echo "═══════════════════════════════════════════"

curl -sf http://localhost:9090/-/healthy > /dev/null && echo "  ✅ Prometheus" || echo "  ❌ Prometheus"
curl -sf http://localhost:9100/metrics > /dev/null && echo "  ✅ Node Exporter" || echo "  ❌ Node Exporter"
curl -sf http://localhost:3000/api/health > /dev/null && echo "  ✅ Grafana" || echo "  ❌ Grafana"
curl -sf http://localhost:3100/ready > /dev/null && echo "  ✅ Loki" || echo "  ❌ Loki"
curl -sf http://localhost:9093/-/healthy > /dev/null && echo "  ✅ AlertManager" || echo "  ❌ AlertManager"

### Service Commands

# Restart a specific service
sudo systemctl restart prometheus
sudo systemctl restart grafana-server
sudo systemctl restart loki
sudo systemctl restart promtail

# View service logs
sudo journalctl -u prometheus -f --no-pager
sudo journalctl -u loki -n 50 --no-pager
sudo journalctl -u promtail -f --no-pager

# Stop all monitoring (if needed)
sudo systemctl stop prometheus node_exporter grafana-server loki promtail alertmanager

---

## Part 13: Troubleshooting

### Common Issues and Solutions

#### Issue 1: Loki shows "activating" or won't start

Error: compactor.delete-request-store should be configured

Solution: The config above includes delete_request_store: filesystem. If still having issues:

# Check Loki config
/usr/local/bin/loki -config.file=/etc/loki/loki-config.yml -verify-config

# Check logs
sudo journalctl -u loki -n 50 --no-pager

# Fix permissions
sudo mkdir -p /var/lib/loki/{chunks,rules,compactor}
sudo chmod 755 /var/lib/loki

#### Issue 2: No logs in Grafana/Loki

Solution:

# Check if promtail is reading files
sudo journalctl -u promtail -n 30 --no-pager | grep -i "target"

# Verify log files exist
ls -la /home/ubuntu/newsportal/logs/
ls -la /var/log/nginx/

# Create log file if missing
sudo mkdir -p /home/ubuntu/newsportal/logs
sudo touch /home/ubuntu/newsportal/logs/app.log
sudo chown ubuntu:ubuntu /home/ubuntu/newsportal/logs/app.log

# Test Loki query
curl -G 'http://localhost:3100/loki/api/v1/query' \
  --data-urlencode 'query={job="django"}' \
  --data-urlencode 'limit=5'

#### Issue 3: Prometheus targets not UP

# Check targets
curl http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E "(job|health)"

# Check if services are listening
sudo netstat -tlnp | grep -E "(9090|9100|8000)"

#### Issue 4: Grafana "Too many redirects"

Solution: Access Grafana directly via port 3000, not through Nginx proxy:

http://YOUR_EC2_IP:3000

#### Issue 5: Permission denied for log files

sudo chown -R ubuntu:ubuntu /home/ubuntu/newsportal/logs
sudo chmod 755 /home/ubuntu/newsportal/logs
sudo chmod 644 /home/ubuntu/newsportal/logs/*.log 2>/dev/null

---

## Quick Reference

### URLs

| Service | URL |
|---------|-----|
| NewsPortal | http://YOUR_EC2_IP |
| Grafana | http://YOUR_EC2_IP:3000 |
| Prometheus | http://YOUR_EC2_IP:9090 |
| AlertManager | http://YOUR_EC2_IP:9093 |

### Default Credentials

| Service | Username | Password |
|---------|----------|----------|
| Grafana | admin | admin123 |
| Django Admin | (created during setup) | (your password) |

### Useful LogQL Queries

# All logs from a specific job
{job="django"}

# Error logs only
{job="django"} |= "ERROR"

# Multiple conditions
{job="nginx"} |~ "5[0-9]{2}" | json

# Exclude debug logs
{job="django"} != "DEBUG"

### Useful PromQL Queries

# CPU Usage
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory Usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk Usage
(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100

# Django Request Rate
rate(django_http_requests_total_by_method_total[5m])

# Django Error Rate (5xx)
rate(django_http_responses_total_by_status_view_method_total{status=~"5.."}[5m])

---

## Support

If you encounter any issues:

1. Check service status: sudo systemctl status <service-name>
2. View logs: sudo journalctl -u <service-name> -f
3. Verify configs: Use the verify commands mentioned above

---

**Author:** Salauddin
**Role:** Software Developer

**Happy Monitoring! 🚀**