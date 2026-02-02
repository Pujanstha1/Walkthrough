# Prometheus Metrics Collection Architecture

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [How Metrics Flow from Node Exporter to Prometheus](#how-metrics-flow-from-node-exporter-to-prometheus)
- [Architecture Diagram](#architecture-diagram)
- [Configuring Prometheus to Scrape Node Exporter](#configuring-prometheus-to-scrape-node-exporter)
- [Setting Up Nginx Reverse Proxy](#setting-up-nginx-reverse-proxy)
- [Configuring DNS in Route 53](#configuring-dns-in-route-53)
- [Installing SSL Certificates with Certbot](#installing-ssl-certificates-with-certbot)
- [Accessing Prometheus and Grafana](#accessing-prometheus-and-grafana)
- [Verification and Testing](#verification-and-testing)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

In a typical Prometheus monitoring setup, the architecture follows a **pull-based model** where:

1. **Node Exporter** runs on target EC2 instances and exposes system metrics on port 9100
2. **Prometheus Server** periodically scrapes (pulls) these metrics and stores them in its time-series database
3. **Grafana** connects to Prometheus to visualize the metrics through dashboards
4. **Nginx** acts as a reverse proxy to provide secure HTTPS access with authentication
5. **Route 53** provides DNS resolution for user-friendly domain access

---

## How Metrics Flow from Node Exporter to Prometheus

### The Pull-Based Model

Unlike traditional monitoring systems that push metrics to a central server, Prometheus uses a **pull-based approach**:

1. **Metrics Exposure**: Node Exporter exposes metrics at `http://<instance-ip>:9100/metrics`
2. **Scrape Configuration**: Prometheus is configured with target endpoints to scrape
3. **Periodic Scraping**: Prometheus pulls metrics from each target at defined intervals (default: 15 seconds)
4. **Time-Series Storage**: Scraped metrics are stored in Prometheus's embedded time-series database
5. **Query and Visualization**: Grafana queries Prometheus using PromQL to create dashboards

### Key Components

| Component | Role | Port | Purpose |
|-----------|------|------|---------|
| **Node Exporter** | Metrics Exporter | 9100 | Exposes system metrics (CPU, memory, disk, network) |
| **Prometheus Server** | Metrics Collector | 9090 | Scrapes and stores time-series data |
| **Grafana** | Visualization | 3000 | Creates dashboards and visualizations |
| **Nginx** | Reverse Proxy | 80/443 | Provides HTTPS and authentication |
| **Route 53** | DNS Service | 53 | Maps domain names to server IP |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          MONITORING INFRASTRUCTURE                      │
└─────────────────────────────────────────────────────────────────────────┘
                                 Internet
                                    │
                                    │ DNS Resolution
                                    ▼
                        ┌───────────────────────┐
                        │     Route 53 (AWS)    │
                        ├───────────────────────┤
                        │ prom.tekbay.click     │────► Prometheus Server
                        │ grafana.tekbay.click  │────► Grafana Server
                        └───────────────────────┘
                                    │
                                    │ HTTPS (443)
                                    ▼
        ┌───────────────────────────────────────────────────────────┐
        │         PROMETHEUS & GRAFANA SERVER                       │
        │         (prom.tekbay.click)                               │
        ├───────────────────────────────────────────────────────────┤
        │                                                           │
        │  ┌────────────────────────────────────────────────────┐   │
        │  │            Nginx Reverse Proxy                     │   │
        │  │              (Port 80/443)                         │   │
        │  ├────────────────────────────────────────────────────┤   │
        │  │  • SSL/TLS Termination (Certbot)                   │   │
        │  │  • Basic Authentication                            │   │
        │  │  • prom.tekbay.click → localhost:9090              │   │
        │  │  • grafana.tekbay.click → localhost:3000           │   │
        │  └────────────┬─────────────────────┬─────────────────┘   │
        │               │                     │                     │
        │               ▼                     ▼                     │
        │   ┌──────────────────────┐  ┌──────────────────────┐      │
        │   │   Prometheus         │  │     Grafana          │      │
        │   │   (Port 9090)        │  │     (Port 3000)      │      │
        │   ├──────────────────────┤  ├──────────────────────┤      │
        │   │ • Time-Series DB     │◄─┤ • Dashboards         │      │
        │   │ • PromQL Engine      │  │ • Alerts             │      │
        │   │ • Scrape Targets     │  │ • Visualizations     │      │
        │   │ • Alertmanager       │  │ • PromQL Queries     │      │
        │   └──────────┬───────────┘  └──────────────────────┘      │
        │              │                                            │
        │              │ Scrapes metrics every 15s                  │
        │              │ (Pull-based model)                         │
        └──────────────┼────────────────────────────────────────────┘
                       │
                       │ HTTP GET /metrics
                       │ Port 9100
                       │
        ┌──────────────┴──────────────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼       
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  EC2 Instance 1  │    │  EC2 Instance 2  │    │  EC2 Instance N  │
│  (EIP: X.X.X.X)  │    │  (EIP: Y.Y.Y.Y)  │    │  (EIP: Z.Z.Z.Z)  │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│  Node Exporter   │    │  Node Exporter   │    │  Node Exporter   │
│    Port: 9100    │    │    Port: 9100    │    │    Port: 9100    │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│ Exposes Metrics: │    │ Exposes Metrics: │    │ Exposes Metrics: │
│ • CPU Usage      │    │ • CPU Usage      │    │ • CPU Usage      │
│ • Memory Usage   │    │ • Memory Usage   │    │ • Memory Usage   │
│ • Disk I/O       │    │ • Disk I/O       │    │ • Disk I/O       │
│ • Network Stats  │    │ • Network Stats  │    │ • Network Stats  │
│ • System Load    │    │ • System Load    │    │ • System Load    │
│ • Uptime         │    │ • Uptime         │    │ • Uptime         │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

### Data Flow Explanation

```
Step 1: Metrics Exposure
─────────────────────────
EC2 Instances run Node Exporter → Exposes metrics at http://<EIP>:9100/metrics

Step 2: Scrape Configuration
─────────────────────────────
Prometheus reads /etc/prometheus/prometheus.yaml → Identifies scrape targets

Step 3: Metrics Collection (Pull)
──────────────────────────────────
Prometheus → HTTP GET http://<EIP>:9100/metrics every 15 seconds

Step 4: Storage
───────────────
Prometheus → Stores time-series data in local database (/var/lib/prometheus/)

Step 5: Visualization
─────────────────────
Grafana → Queries Prometheus using PromQL → Displays dashboards

Step 6: User Access
───────────────────
User → https://prom.tekbay.click → Nginx → Prometheus (localhost:9090)
User → https://grafana.tekbay.click → Nginx → Grafana (localhost:3000)
```

---

## Configuring Prometheus to Scrape Node Exporter

### Step 1: Edit Prometheus Configuration

SSH into your Prometheus server and edit the configuration file:

```bash
sudo nano /etc/prometheus/prometheus.yaml
```

### Step 2: Add Scrape Targets

Add your Node Exporter targets to the configuration:

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  
  # Scrape Prometheus itself
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          instance: "prometheus-server"
          environment: "production"

  # Scrape Node Exporter from multiple EC2 instances
  - job_name: "node_exporter"
    static_configs:
      # First EC2 Instance
      - targets: ["54.123.45.67:9100"]
        labels:
          instance: "web-server-1"
          environment: "production"
          region: "us-east-1"
      
      # Second EC2 Instance
      - targets: ["54.234.56.78:9100"]
        labels:
          instance: "web-server-2"
          environment: "production"
          region: "us-east-1"
      
      # Third EC2 Instance
      - targets: ["54.345.67.89:9100"]
        labels:
          instance: "database-server"
          environment: "production"
          region: "us-west-2"
      
    ### Scrape Node Exporter from all EC2 instances (For Multiple instaces, this can be done as well!! instead of the 3 above)
    # - job_name: "node_exporter"
    #     static_configs:
    #     - targets:
    #           - "54.123.45.67:9100"
    #           - "54.234.56.78:9100"
    #           - "54.345.67.89:9100"
    #           - "54.456.78.90:9100"
    #           - "54.567.89.01:9100"
    #        labels:
    #           environment: "production"
```

**Configuration Breakdown:**

- **`job_name`**: Logical grouping of targets (appears as a label in metrics)
- **`targets`**: List of `<host>:port` endpoints to scrape
- **`labels`**: Custom labels attached to all metrics from this target
  - `instance`: Human-readable name for the server
  - `environment`: Deployment environment (production, staging, dev)
  - `region`: AWS region or datacenter location

### Step 3: Validate Configuration

Before restarting Prometheus, validate the configuration syntax:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Expected output:
```
Checking /etc/prometheus/prometheus.yml
  SUCCESS: 3 rule files found
  SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

### Step 4: Restart Prometheus

Restart Prometheus to apply the new configuration:

```bash
sudo systemctl restart prometheus
```

Check the service status:

```bash
sudo systemctl status prometheus
```

### Step 5: Verify Targets

Open Prometheus UI and check targets:

```
http://<PROMETHEUS_SERVER_IP>:9090/targets
```

You should see all configured targets with their status:
- **UP** (green): Successfully scraping metrics
- **DOWN** (red): Cannot reach the target

---

## Setting Up Nginx Reverse Proxy

Nginx will act as a reverse proxy to provide:
- **HTTPS encryption** via Let's Encrypt SSL certificates
- **Basic authentication** to protect access
- **Domain-based routing** (prom.tekbay.click → Prometheus, grafana.tekbay.click → Grafana)

### Step 1: Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

Start and enable Nginx:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Step 2: Create Nginx Configuration for Prometheus

Create a new server block for Prometheus:

```bash
sudo nano /etc/nginx/sites-available/prom.tekbay.click
```

Add the following configuration:

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name prom.tekbay.click;

    # Redirect all HTTP traffic to HTTPS
    return 301 https://$server_name$request_uri;
}

# HTTPS server block for Prometheus
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name prom.tekbay.click;

    # SSL certificate paths (will be added by Certbot)
    # ssl_certificate /etc/letsencrypt/live/prom.tekbay.click/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/prom.tekbay.click/privkey.pem;
    # include /etc/letsencrypt/options-ssl-nginx.conf;
    # ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Access and error logs
    access_log /var/log/nginx/prometheus-access.log;
    error_log /var/log/nginx/prometheus-error.log;

    # Basic Authentication
    auth_basic "Prometheus Access";
    auth_basic_user_file /etc/nginx/.htpasswd;

    # Proxy settings
    location / {
        proxy_pass http://localhost:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support (for real-time updates)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### Step 3: Create Nginx Configuration for Grafana

Create a new server block for Grafana:

```bash
sudo nano /etc/nginx/sites-available/grafana.tekbay.click
```

Add the following configuration:

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name grafana.tekbay.click;

    # Redirect all HTTP traffic to HTTPS
    return 301 https://$server_name$request_uri;
}

# HTTPS server block for Grafana
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name grafana.tekbay.click;

    # SSL certificate paths (will be added by Certbot)
    # ssl_certificate /etc/letsencrypt/live/grafana.tekbay.click/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/grafana.tekbay.click/privkey.pem;
    # include /etc/letsencrypt/options-ssl-nginx.conf;
    # ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Access and error logs
    access_log /var/log/nginx/grafana-access.log;
    error_log /var/log/nginx/grafana-error.log;

    # Proxy settings
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support (required for Grafana Live)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # API endpoint (optional, for better performance)
    location /api/ {
        proxy_pass http://localhost:3000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Step 4: Create Basic Authentication for Prometheus

Install Apache utilities to create password file:

```bash
sudo apt install apache2-utils -y
```

Create a password file with a username (replace `admin` with your desired username):

```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

You'll be prompted to enter and confirm a password.

To add additional users:

```bash
sudo htpasswd /etc/nginx/.htpasswd another_user
```

Verify the file:

```bash
cat /etc/nginx/.htpasswd
```

Output:
```
admin:$apr1$abc123$xyz...
```

### Step 5: Enable the Nginx Configurations

Create symbolic links to enable the sites:

```bash
sudo ln -s /etc/nginx/sites-available/prom.tekbay.click /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/grafana.tekbay.click /etc/nginx/sites-enabled/
```

### Step 6: Test Nginx Configuration

Before restarting Nginx, test the configuration for syntax errors:

```bash
sudo nginx -t
```

Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Step 7: Restart Nginx

Restart Nginx to apply the changes:

```bash
sudo systemctl restart nginx
```

Check status:

```bash
sudo systemctl status nginx
```

---

## Configuring DNS in Route 53

### Step 1: Log into AWS Console

1. Navigate to **Route 53** service
2. Click on **Hosted Zones**
3. Select your hosted zone: `tekbay.click`

### Step 2: Create A Record for Prometheus

1. Click **Create Record**
2. Fill in the details:
   - **Record name**: `prom`
   - **Record type**: `A - Routes traffic to an IPv4 address and some AWS resources`
   - **Value**: Enter your Prometheus/Grafana server's **Elastic IP** or **Public IP**
   - **TTL (seconds)**: `300` (5 minutes)
   - **Routing policy**: `Simple routing`
3. Click **Create records**

This creates: `prom.tekbay.click` → `<Your-Server-IP>`

### Step 3: Create A Record for Grafana

1. Click **Create Record** again
2. Fill in the details:
   - **Record name**: `grafana`
   - **Record type**: `A - Routes traffic to an IPv4 address and some AWS resources`
   - **Value**: Enter the **same IP address** as Prometheus (since they're on the same server)
   - **TTL (seconds)**: `300`
   - **Routing policy**: `Simple routing`
3. Click **Create records**

This creates: `grafana.tekbay.click` → `<Your-Server-IP>`

### Step 4: Verify DNS Propagation

Wait a few minutes for DNS propagation, then test:

```bash
# Test Prometheus domain
dig prom.tekbay.click +short
nslookup prom.tekbay.click

# Test Grafana domain
dig grafana.tekbay.click +short
nslookup grafana.tekbay.click
```

Both should return your server's IP address.

### Alternative: Using AWS CLI

```bash
# Get your hosted zone ID
aws route53 list-hosted-zones --query "HostedZones[?Name=='tekbay.click.'].Id" --output text

# Create Prometheus A record
aws route53 change-resource-record-sets \
  --hosted-zone-id <ZONE_ID> \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "prom.tekbay.click",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "<YOUR_SERVER_IP>"}]
      }
    }]
  }'

# Create Grafana A record
aws route53 change-resource-record-sets \
  --hosted-zone-id <ZONE_ID> \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "grafana.tekbay.click",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "<YOUR_SERVER_IP>"}]
      }
    }]
  }'
```

---

## Installing SSL Certificates with Certbot

Certbot is a free, automated tool to obtain and renew SSL/TLS certificates from Let's Encrypt.

### Step 1: Install Certbot

```bash
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
```

Create a symbolic link to make `certbot` accessible:

```bash
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Verify installation:

```bash
certbot --version
```

### Step 2: Obtain SSL Certificate for Prometheus

Run Certbot with the Nginx plugin:

```bash
sudo certbot --nginx -d prom.tekbay.click
```

**Interactive prompts:**

1. **Email address**: Enter your email for renewal notifications
2. **Terms of Service**: Agree to the terms (yes)
3. **Share email**: Choose whether to share your email with EFF (optional)
4. **Redirect HTTP to HTTPS**: Select option `2` (recommended)

Certbot will:
- Validate domain ownership
- Obtain the SSL certificate
- Automatically modify your Nginx configuration
- Set up auto-renewal

Expected output:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/prom.tekbay.click/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/prom.tekbay.click/privkey.pem
This certificate expires on 2026-05-03.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.
```

### Step 3: Obtain SSL Certificate for Grafana

```bash
sudo certbot --nginx -d grafana.tekbay.click
```

Follow the same prompts as above.

### Step 4: Verify SSL Configuration

Check your Nginx configuration to see the SSL directives added by Certbot:

```bash
sudo cat /etc/nginx/sites-available/prom.tekbay.click
```

You should see lines like:

```nginx
ssl_certificate /etc/letsencrypt/live/prom.tekbay.click/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/prom.tekbay.click/privkey.pem;
include /etc/letsencrypt/options-ssl-nginx.conf;
ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
```

### Step 5: Test SSL Certificates

```bash
# Test Prometheus SSL
curl -I https://prom.tekbay.click

# Test Grafana SSL
curl -I https://grafana.tekbay.click
```

Or use an online tool: [SSL Labs Server Test](https://www.ssllabs.com/ssltest/)

### Step 6: Set Up Auto-Renewal

Certbot automatically sets up a systemd timer for renewal. Verify it:

```bash
sudo systemctl list-timers | grep certbot
```

Test renewal (dry run):

```bash
sudo certbot renew --dry-run
```

Expected output:
```
Congratulations, all simulated renewals succeeded
```

### Certificate Renewal Process

Let's Encrypt certificates expire after **90 days**. Certbot automatically renews certificates when they have **30 days or less** remaining.

Manual renewal (if needed):

```bash
sudo certbot renew
```

Renewal will also reload Nginx automatically.

---

## Accessing Prometheus and Grafana

### Access Prometheus

1. Open your browser and navigate to:
   ```
   https://prom.tekbay.click
   ```

2. You'll be prompted for authentication (Nginx Basic Auth):
   - **Username**: `admin` (or whatever you configured)
   - **Password**: [your password]

3. Once logged in, you'll see the Prometheus UI

4. Navigate to **Status → Targets** to verify all Node Exporters are being scraped:
   ```
   https://prom.tekbay.click/targets
   ```

### Access Grafana

1. Open your browser and navigate to:
   ```
   https://grafana.tekbay.click
   ```

2. Log in with Grafana credentials (default):
   - **Username**: `admin`
   - **Password**: `admin` (you'll be prompted to change it on first login)

3. Add Prometheus as a data source:
   - Go to **Configuration → Data Sources**
   - Click **Add data source**
   - Select **Prometheus**
   - **URL**: `http://localhost:9090`
   - Click **Save & Test**

4. Import a Node Exporter dashboard:
   - Go to **Dashboards → Import**
   - Enter dashboard ID: **1860**
   - Select Prometheus data source
   - Click **Import**

---

## Verification and Testing

### 1. Check Prometheus Targets

Visit: `https://prom.tekbay.click/targets`

All targets should show:
- **State**: UP (green)
- **Last Scrape**: Recent timestamp
- **Scrape Duration**: < 1 second

### 2. Query Metrics in Prometheus

Go to **Graph** tab and try these queries:

**CPU Usage:**
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Memory Usage:**
```promql
100 * (1 - ((node_memory_MemAvailable_bytes) / (node_memory_MemTotal_bytes)))
```

**Disk Usage:**
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"} * 100) / node_filesystem_size_bytes{mountpoint="/",fstype!="rootfs"})
```

### 3. Test Node Exporter Endpoints

```bash
# Test from Prometheus server
curl http://54.123.45.67:9100/metrics | head -20

# Test from local machine
curl http://<EIP>:9100/metrics | grep node_cpu
```

### 4. Check Nginx Logs

```bash
# Prometheus access logs
sudo tail -f /var/log/nginx/prometheus-access.log

# Grafana access logs
sudo tail -f /var/log/nginx/grafana-access.log

# Check for errors
sudo tail -f /var/log/nginx/prometheus-error.log
sudo tail -f /var/log/nginx/grafana-error.log
```

### 5. Verify SSL Certificates

```bash
# Check certificate expiry
sudo certbot certificates

# Output shows:
# Certificate Name: prom.tekbay.click
#   Expiry Date: 2026-05-03 12:00:00+00:00 (VALID: 89 days)
```

---

## Troubleshooting

### Prometheus Can't Scrape Targets

**Symptom**: Targets show "DOWN" status

**Solutions**:

1. **Check Node Exporter is running on target:**
   ```bash
   ssh ubuntu@<target-ip>
   sudo systemctl status node_exporter
   ```

2. **Verify port 9100 is open in security group:**
   ```bash
   aws ec2 describe-security-groups --group-ids sg-xxxxx \
     --query 'SecurityGroups[0].IpPermissions[?ToPort==`9100`]'
   ```

3. **Test connectivity from Prometheus server:**
   ```bash
   curl http://<target-ip>:9100/metrics
   telnet <target-ip> 9100
   ```

4. **Check Prometheus logs:**
   ```bash
   sudo journalctl -u prometheus -f
   ```

### Nginx 502 Bad Gateway

**Symptom**: Can't access Prometheus or Grafana through domain

**Solutions**:

1. **Check if services are running:**
   ```bash
   sudo systemctl status prometheus
   sudo systemctl status grafana-server
   ```

2. **Verify they're listening on correct ports:**
   ```bash
   sudo netstat -tlnp | grep 9090  # Prometheus
   sudo netstat -tlnp | grep 3000  # Grafana
   ```

3. **Check Nginx error logs:**
   ```bash
   sudo tail -f /var/log/nginx/error.log
   ```

4. **Test proxy connection:**
   ```bash
   curl http://localhost:9090  # Prometheus
   curl http://localhost:3000  # Grafana
   ```

### SSL Certificate Issues

**Symptom**: Certificate warnings or errors

**Solutions**:

1. **Verify DNS is pointing to correct IP:**
   ```bash
   dig prom.tekbay.click +short
   ```

2. **Check certificate files exist:**
   ```bash
   sudo ls -l /etc/letsencrypt/live/prom.tekbay.click/
   ```

3. **Test SSL configuration:**
   ```bash
   sudo nginx -t
   ```

4. **Re-obtain certificate:**
   ```bash
   sudo certbot delete --cert-name prom.tekbay.click
   sudo certbot --nginx -d prom.tekbay.click
   ```

### Authentication Not Working

**Symptom**: Can't log in despite correct credentials

**Solutions**:

1. **Verify .htpasswd file exists:**
   ```bash
   sudo cat /etc/nginx/.htpasswd
   ```

2. **Recreate password:**
   ```bash
   sudo htpasswd -c /etc/nginx/.htpasswd admin
   ```

3. **Check Nginx configuration syntax:**
   ```bash
   sudo nginx -t
   ```

4. **Clear browser cache and cookies**

### Grafana Can't Connect to Prometheus

**Symptom**: "Bad Gateway" or connection errors in Grafana

**Solutions**:

1. **Use localhost URL in Grafana:**
   - Data source URL should be: `http://localhost:9090`
   - NOT: `https://prom.tekbay.click`

2. **Check Prometheus is accessible:**
   ```bash
   curl http://localhost:9090/api/v1/query?query=up
   ```

3. **Verify no firewall blocking:**
   ```bash
   sudo ufw status
   ```

---

## Security Best Practices

### 1. Restrict Port 9100 Access

Update your EC2 security groups to only allow Prometheus server IP:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 9100 \
  --cidr <PROMETHEUS_SERVER_IP>/32
```

### 2. Enable Firewall (UFW)

```bash
# On Prometheus/Grafana server
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw allow 9100/tcp    # Node Exporter (if self-monitoring)
sudo ufw enable
```

### 3. Use Strong Passwords

```bash
# Generate strong password
openssl rand -base64 24
```

### 4. Regular Updates

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Update Prometheus
# Download latest version and replace binary

# Update Grafana
sudo apt update
sudo apt upgrade grafana
```

### 5. Monitor SSL Certificate Expiry

Set up alerts in Prometheus to monitor certificate expiry:

```yaml
- alert: SSLCertificateExpiry
  expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "SSL certificate expiring soon"
```

---

## Summary

You now have a complete monitoring infrastructure with:

✅ **Node Exporter** running on EC2 instances exposing system metrics  
✅ **Prometheus** scraping metrics and storing time-series data  
✅ **Grafana** visualizing metrics through dashboards  
✅ **Nginx** providing HTTPS and authentication  
✅ **Route 53** DNS resolution for user-friendly domains  
✅ **Let's Encrypt SSL** certificates with auto-renewal  

### Quick Reference

| Service | URL | Port | Purpose |
|---------|-----|------|---------|
| Prometheus | https://prom.tekbay.click | 9090 | Metrics collection & storage |
| Grafana | https://grafana.tekbay.click | 3000 | Visualization & dashboards |
| Node Exporter | http://`<EIP>`:9100/metrics | 9100 | System metrics exposure |

### Next Steps

1. **Create custom dashboards** in Grafana for your specific use cases
2. **Set up alerting rules** in Prometheus for critical metrics
3. **Configure Alertmanager** to send notifications (email, Slack, PagerDuty)
4. **Add more exporters** (MySQL, Redis, Nginx) as needed
5. **Implement service discovery** for dynamic environments
6. **Set up long-term storage** using Thanos or Cortex for metrics retention

---

**Documentation Last Updated**: February 2026
