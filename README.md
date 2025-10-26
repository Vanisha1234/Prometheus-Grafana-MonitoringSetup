Prometheus-Grafana-MonitoringSetup
Project Overview
This project demonstrates a complete infrastructure monitoring setup using Prometheus, Node Exporter, and Grafana, configured natively on Linux.
It provides real-time insights into CPU, memory, and disk utilization for a virtual machine and automatically sends alerts to Microsoft Teams when resource usage crosses a defined threshold.

Tech Stack
Monitoring: Prometheus, Node Exporter
Visualization & Alerting: Grafana
Notification Channel: Microsoft Teams Webhook
Environment: Ubuntu (WSL) + Linux VM

Key Components
Prometheus – Collects system metrics from Node Exporter and stores time-series data.
Node Exporter – Runs on the target VM to expose hardware and OS-level metrics.
Grafana – Visualizes metrics through custom dashboards and manages alerting rules.
Microsoft Teams Integration – Configured via an incoming webhook to send real-time notifications when CPU, memory, or disk usage exceeds 70%.

