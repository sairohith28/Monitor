# Prometheus, Grafana, and Alertmanager Setup Guide on Ubuntu

This guide walks you through the installation and setup of **Prometheus**, **Grafana**, and **Alertmanager** on Ubuntu. You will also learn how to monitor a Flask API and configure alert rules for performance and memory usage.

## Prerequisites

Make sure your system is updated:

```bash
sudo apt update
```

## Step 1: Install Prometheus

```bash
sudo apt install -y prometheus
```

Enable and start the Prometheus service:

```bash
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

## Step 2: Install Grafana

### Install Dependencies

```bash
sudo apt install -y apt-transport-https software-properties-common wget
```

### Add Grafana GPG Key

```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

### Add Grafana Repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

### Install Grafana

```bash
sudo apt update
sudo apt install -y grafana
```

### Start and Enable Grafana

```bash
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

### Access Grafana

Grafana is accessible at: `http://<your-server-ip>:3000`

* **Username**: `admin`
* **Password**: `admin` (you'll be prompted to change on first login)

## Step 3: Install Alertmanager

### Download Alertmanager

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
```

### Extract and Install

```bash
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64
sudo mv alertmanager /usr/local/bin/
sudo mv amtool /usr/local/bin/
```

### Configure Alertmanager

```bash
sudo mkdir /etc/alertmanager
sudo mv alertmanager.yml /etc/alertmanager/
```

### Create Systemd Service

```bash
sudo nano /etc/systemd/system/alertmanager.service
```

Paste the following content:

```ini
[Unit]
Description=Alertmanager for Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager
Restart=always

[Install]
WantedBy=multi-user.target
```

### Create User and Data Directory

```bash
sudo useradd -M -s /bin/false prometheus
sudo mkdir /var/lib/alertmanager
sudo chown prometheus:prometheus /var/lib/alertmanager
```

### Start and Enable Alertmanager

```bash
sudo systemctl daemon-reload
sudo systemctl start alertmanager
sudo systemctl enable alertmanager
```

### Verify

Alertmanager is accessible at: `http://<your-server-ip>:9093`

## Step 4: Integrate with Flask API

### Install Prometheus Exporter

```bash
pip install prometheus_flask_exporter
```

### Modify Flask Code

```python
from flask import Flask
from prometheus_flask_exporter import PrometheusMetrics

app = Flask(__name__)
metrics = PrometheusMetrics(app)

@app.route('/')
def home():
    return "Hello, World!"

if __name__ == '__main__':
    app.run()
```

## Step 5: Configure Prometheus

### Edit Prometheus Configuration

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'flask-api'
    static_configs:
      - targets: ['localhost:5000']
```

### Restart Prometheus

```bash
sudo systemctl restart prometheus
```

## Step 6: Configure Alerts

### Alertmanager Configuration `/etc/alertmanager/alertmanager.yml`

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'email-notification'
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h

receivers:
  - name: 'email-notification'
    email_configs:
      - to: 'your_email@example.com'
        from: 'alertmanager@example.com'
        smtp_smarthost: 'smtp.example.com:587'
        smtp_auth_username: 'your_smtp_username'
        smtp_auth_password: 'your_smtp_password'
        send_resolved: true
```

### Define Alerting Rules `/etc/prometheus/alert.rules.yml`

```yaml
groups:
  - name: flask-api-alerts
    rules:
      - alert: HighRequestLatency
        expr: flask_http_request_duration_seconds{status="200"} > 0.5
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High request latency for Flask API"
          description: "Request latency exceeded 0.5 seconds for API endpoint {{ $labels.path }}"

      - alert: HighMemoryUsage
        expr: process_resident_memory_bytes > 100000000
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage for Flask API"
          description: "Memory usage exceeded 100 MB for Flask API"
```

### Link Rules in Prometheus Config

```yaml
rule_files:
  - /etc/prometheus/alert.rules.yml
```

### Restart Services

```bash
sudo systemctl restart prometheus alertmanager
```

## Step 7: Visualize with Grafana

* Access Grafana at `http://localhost:3000`
* Add Prometheus as a data source: `http://localhost:9090`
* Example Query:

```promQL
rate(flask_http_request_duration_seconds_sum[1m]) / rate(flask_http_request_duration_seconds_count[1m])
```

## Step 8: Configure Storage in Prometheus

### Update Prometheus Config

```yaml
storage:
  tsdb:
    path: /var/lib/prometheus/data
    retention: 15d
```

### Ensure Directory and Permissions

```bash
sudo mkdir -p /var/lib/prometheus/data
sudo chown -R prometheus:prometheus /var/lib/prometheus/data
```

### Restart Prometheus

```bash
sudo systemctl restart prometheus
```

---

This setup enables performance and memory monitoring of Flask APIs, supports email alerts on threshold breaches, and offers visualization via Grafana.
