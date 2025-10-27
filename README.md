# Prometheus-Grafana Monitoring Setup

This project demonstrates a complete infrastructure monitoring setup using **Prometheus**, **Node Exporter**, and **Grafana**, configured natively on Linux. It provides real-time insights into **CPU**, **memory**, and **disk utilization** for a virtual machine and automatically sends alerts to **Microsoft Teams** when resource usage crosses a defined threshold.

I set up this monitoring project in my workplace to proactively track system performance and prevent last-minute issues such as high CPU utilization, low disk space, and memory exhaustion.

---

## Table of Contents
1. [Project Overview](#prometheus-grafana-monitoring-setup)
2. [Tech Stack](#tech-stack)
3. [Key Components](#key-components-and-flow)
4. [Project Environment Setup](#project-environment-setup)
   - [Prometheus Setup on Linux](#prometheus-setup-on-linux)
   - [Grafana Setup on Linux](#grafana-setup-on-linux)
   - [Node Exporter Setup on Target Linux VM](#node-exporter-setup-on-target-linux-vm)
5. [Configurations & Integrations](#configurations--integrations)
   - [Connect Grafana with Prometheus](#connect-grafana-with-prometheus)
   - [Import Node Exporter Dashboard](#import-node-exporter-dashboard)
6. [Alert Notifications via Teams](#alert-notifications-via-teams)
   - [Create a Microsoft Teams Incoming Webhook](#create-a-microsoft-teams-incoming-webhook)
   - [Add a Microsoft Teams Notification Channel in Grafana](#add-a-microsoft-teams-notification-channel-in-grafana)
   - [Create an Alert Rule for Core Components](#create-an-alert-rule-for-core-components-like-cpu-memory-and-disk)

---

## Tech Stack
- **Monitoring:** Prometheus, Node Exporter  
- **Visualization & Alerting:** Grafana  
- **Notification Channel:** Microsoft Teams Webhook  
- **Environment:** Ubuntu (WSL) + Linux VM  

---

## Key Components and Flow
- **Node Exporter:**
  1. Installed on the target virtual machine being monitor.
     
  2. Exposes system-level metrics (CPU, memory, disk, network) & makes them available at an HTTP endpoint:
 ```bash
 http://<node-ip>:9100/metrics
 ```

- **Prometheus:** 
  1. Prometheus periodically scrapes metrics from Node Exporter.
  2. Inside Prometheus:
     
     > ***a. Retrieval module:*** Pulls (retrieves) metrics data. \
     > ***b. Storage module:*** Stores it in the Time Series Database (TSDB). \
     > ***c. HTTP Server module:*** Accepts PromQL queries.

- **Grafana:**
  1. Grafana connects to Prometheus as a data source.
     
  2. When we build dashboards in Grafana, the graphs are based on PromQL queries that Grafana sends to the Prometheus HTTP API in order to retrieve data.
     
  3. Prometheus returns the requested data, and Grafana visualizes metrics through custom dashboards.
     
- **Microsoft Teams Integration:**
  1. Grafana manages alerts natively and we can define threshold-based alert rules.
     
  2. Alerts are sent to Microsoft Teams via an incoming webhook for real-time updates when CPU, memory, or disk usage exceeds defined threshold..

---

# Project Environment Setup
### Prometheus Setup on Linux:
> Update and install dependencies:
```bash
sudo apt update && sudo apt install -y wget tar
```
> Download and install Prometheus:
```bash
cd /opt
sudo apt update -y
sudo apt install -y wget tar
sudo wget https://github.com/prometheus/prometheus/releases/download/v3.7.1/prometheus-3.7.1.linux-amd64.tar.gz
sudo tar xvf prometheus-3.7.1.linux-amd64.tar.gz
sudo mv prometheus-3.7.1.linux-amd64 prometheus
```
> Create Prometheus user:
```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```
> Move binaries and create directories:
```bash
sudo mv prometheus/prometheus /usr/local/bin/
sudo mv prometheus/promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```
> Configure Prometheus:
> Create /etc/prometheus/prometheus.yml:
```bash
sudo nano /etc/prometheus/prometheus.yml
```
> Add file content:
```bash
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'vm_node_exporter'
    static_configs:
      - targets: ['192.168.1.150:9100']   # <-- replace with your VM IP
```
In my case the VM IP is: "192.168.126.131:9100"

> Create Prometheus systemd service:
```bash
sudo nano /etc/systemd/system/prometheus.service
```
> Add file content:
```bash
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/opt/prometheus/consoles \
  --web.console.libraries=/opt/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```
> Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```
> Check in browser:
```bash
http://localhost:9090
```

### Grafana Setup on Linux: 
> Add Grafana repo & install:
```bash
sudo apt install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo wget -q -O /usr/share/keyrings/grafana.key https://packages.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
```
> Enable & start:
```bash
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```
> Check in browser:
```bash
http://localhost:3000
```
> Default Login: username: admin / password: admin  


### Node exporter setup on Target Linux VM:
> Install node_exporter:
```bash
cd /opt
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
sudo tar xvf node_exporter-1.9.1.linux-amd64.tar.gz
sudo mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false nodeusr
```
> Check node_exporter version:
```bash
node_exporter --version
```
> Create systemd service:
```bash
sudo nano /etc/systemd/system/node_exporter.service
```
> Add file content:
```bash
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=nodeusr
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
```
> Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```
> Verify:
```bash
curl http://localhost:9100/metrics | head
```
If it shows metrics - it’s working

> Now test from Prom & Grafana VM side, Browse:
```bash
http://192.168.126.131:9100/metrics
```

---

# Configurations & Integrations
### Connect Grafana with Prometheus:
- Go to Grafana → Connections → Data Sources → Add new data source
<img width="1904" height="954" alt="image" src="https://github.com/user-attachments/assets/bf8b5239-125d-4a42-a1e1-acbb10db6e67" />

- Choose Prometheus

> Provide default Prometheus URL:

```bash
http://localhost:9090
```
<img width="1891" height="955" alt="image" src="https://github.com/user-attachments/assets/8fecb73c-47bb-490b-bbbb-a421a8a7c0ec" />

- Save & Test should show following result:

<img width="1486" height="199" alt="image" src="https://github.com/user-attachments/assets/35d0bfee-8016-4311-8ee3-53e1a0bb2bcb" />

### Import Node Exporter Dashboard:
- In Grafana → Dashboards → Import
<img width="1915" height="853" alt="image" src="https://github.com/user-attachments/assets/fb2544b4-7c6e-4ead-a1a1-f69c25c052b8" />

- Enter dashboard ID 1860 (Node Exporter Full) and click on Load

<img width="1905" height="856" alt="image" src="https://github.com/user-attachments/assets/6a831323-cac3-411b-a7eb-169953f0bf29" />

- Select Prometheus data source and click “Import”.

- Switch to your data source from default.

- It will now display live CPU, memory, disk usage, etc. from your VM.

<img width="1900" height="942" alt="image" src="https://github.com/user-attachments/assets/cf036fbb-ed4a-4244-a745-ddbdfda298e9" />

---

# Alert Notifications via Teams
### Create a Microsoft Teams Incoming Webhook:
- Microsoft Teams → team/channel → Edit Connectors

<img width="1882" height="1001" alt="image" src="https://github.com/user-attachments/assets/5a889f71-2336-4774-a5ea-5ffc337c6698" />

- Configure Incoming Webhooks:

<img width="714" height="889" alt="image" src="https://github.com/user-attachments/assets/20aa97c5-046a-4a35-adb2-17fc13887652" />

- Provide a name and select create:

<img width="725" height="863" alt="image" src="https://github.com/user-attachments/assets/87834b29-d2fa-4f53-a0a1-ec16a83d10a5" />

> This generates a Webhook URL which looks like this:
```bash
https://outlook.office.com/webhook/xxxxxx
```

### Add a Microsoft Teams Notification Channel in Grafana:
- In Grafana → Alerting → Contact points

<img width="1909" height="864" alt="image" src="https://github.com/user-attachments/assets/c53daffe-a811-4426-b754-9caa79f313ee" />

- Create a new contact point and provide the Webhook URL generated from Teams

<img width="1905" height="931" alt="image" src="https://github.com/user-attachments/assets/ea652cb9-d841-425e-941e-a045b0b5ea6d" />

- Test the contact point before saving it. If you recieve the following test notification from teams, it is working fine.

<img width="1913" height="907" alt="image" src="https://github.com/user-attachments/assets/1044f14d-e540-47ff-be97-5b2faef0dfe1" />

### Create an Alert Rule for core components like CPU, Memory, and Disk:
- Dashboard → Alerting tab → Create Alert Rule 

**1. Enter alert rule name**
- Enter a name to identify your alert rule.
  
**2. Define query and alert condition**
- Choose your Datasource and click on code
- Provide the PromQL Query which calculated the component usage of all the instances
> For CPU Usage
```bash
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
> For Memory Usage
```bash
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```
> For Disk Usage
```bash
(1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})) * 100
```
> Run the above queries to verify if it displays the metrics or not.
- Expressions
     
<img width="835" height="413" alt="image" src="https://github.com/user-attachments/assets/ebcc7f21-420c-4a7e-8671-19c97d218302" />
   
- Set Expression to A (your PromQL query)
- Condition: IS ABOVE
- Value: 70 (Set the Value as 70 if you want to trigger a notification once the component usage reaches above the defined threshold 70%)
      
**3. Add folder and labels**
- Create a folder and set of labels to organise alert rules.
> Add example labels like:
```bash
severity = critical
type = system
component = cpu
environment = production
```

**4. Set evaluation behavior**
- Define how the alert rule is evaluated.
- Create a new evaluation or group and select an existing one
- Period during which the threshold condition must be met to trigger an alert. If you select 2m (The component will be evaluated every 2 mins and notification       will be triggered if component usage reaches above the defined threshold)
     
**5. Configure notifications**
- Select who should receive a notification when an alert rule fires.
- Select the contact point that we configured (In our case - Teams)
     
**6. Configure notification message**
- Summary (optional)
> Short summary of what happened and why.
> Example:
```bash
High CPU Usage on {{ $labels.instance }}
```
- Description (optional)
> Description of what the alert rule does.
> Example:
```bash
CPU usage has crossed 70% on {{ $labels.instance }} for more than 2 minutes. Please check running processes or workloads on the node.
```

**7. Save Rule & Exit** \
***! Now, Once the component usage exceeds the defined threshold the user will get notified through Teams notification***

  
