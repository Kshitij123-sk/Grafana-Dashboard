# Home Lab Monitoring Stack - Setup Guide

 
STACK OVERVIEW 
===============

Component         Role                              Port
---------         ----                              ----
Prometheus        Metrics collection & storage      9091
Grafana           Visualization & dashboards        3000
Node Exporter     Linux/RHEL host metrics           9100
Windows Exporter  Windows host metrics              9182


Architecture:

  Windows 11 Host (192.168.0.105)
      └── windows_exporter :9182
              │
              ▼
  RHEL 10 VM - rhel10-srv (192.168.8.100)
      ├── Prometheus :9091  <-- scrapes both exporters
      ├── Node Exporter :9100
      └── Grafana :3000  <-- queries Prometheus


===============================
STEP 1 - INSTALL PROMETHEUS
===============================

1. Download and extract Prometheus:

   wget https://github.com/prometheus/prometheus/releases/latest/download/prometheus-*.linux-amd64.tar.gz
   tar xvf prometheus-*.linux-amd64.tar.gz
   sudo mv prometheus-*/prometheus /usr/local/bin/
   sudo mv prometheus-*/promtool /usr/local/bin/

2. Create directories and user:

   sudo useradd --no-create-home --shell /bin/false prometheus
   sudo mkdir /etc/prometheus /var/lib/prometheus

3. Create config file at /etc/prometheus/prometheus.yml:

   global:
     scrape_interval: 15s

   scrape_configs:
     - job_name: 'node_exporter'
       static_configs:
         - targets: ['localhost:9100']

     - job_name: 'windows_exporter'
       static_configs:
         - targets: ['192.168.0.105:9182']

   NOTE: Job name must match exactly what you select in Grafana dashboard
         variable dropdowns later.

4. Create systemd service at /etc/systemd/system/prometheus.service:

   [Unit]
   Description=Prometheus
   After=network.target

   [Service]
   User=prometheus
   ExecStart=/usr/local/bin/prometheus \
     --config.file=/etc/prometheus/prometheus.yml \
     --storage.tsdb.path=/var/lib/prometheus \
     --web.listen-address=:9091
   Restart=always

   [Install]
   WantedBy=multi-user.target

   NOTE: Port 9091 used here because port 9090 was occupied by websm service.
         Change to 9090 if that port is free on your system.

5. Start and enable:

   sudo systemctl daemon-reload
   sudo systemctl enable --now prometheus

6. Verify Prometheus is running:

   curl http://localhost:9091/-/healthy


===============================
STEP 2 - INSTALL NODE EXPORTER (RHEL 10 host metrics)
===============================

1. Download and extract:

   wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-*.linux-amd64.tar.gz
   tar xvf node_exporter-*.linux-amd64.tar.gz
   sudo mv node_exporter-*/node_exporter /usr/local/bin/

2. Create user:

   sudo useradd --no-create-home --shell /bin/false node_exporter

3. Create systemd service at /etc/systemd/system/node_exporter.service:

   [Unit]
   Description=Node Exporter
   After=network.target

   [Service]
   User=node_exporter
   ExecStart=/usr/local/bin/node_exporter
   Restart=always

   [Install]
   WantedBy=multi-user.target

4. Start and enable:

   sudo systemctl daemon-reload
   sudo systemctl enable --now node_exporter

5. Verify metrics are being exposed:

   curl http://localhost:9100/metrics | head -20


===============================
STEP 3 - INSTALL WINDOWS EXPORTER (Windows 11 host)
===============================

1. Download the latest .msi installer from:
   https://github.com/prometheus-community/windows_exporter/releases

2. Run the .msi installer on your Windows machine.
   It installs and starts automatically as a Windows service.

3. Verify by opening this URL in your browser on the Windows machine:
   http://localhost:9182/metrics

4. Allow inbound traffic on port 9182 (run in PowerShell as Administrator):

   New-NetFirewallRule -DisplayName "windows_exporter" -Direction Inbound -Protocol TCP -LocalPort 9182 -Action Allow

5. Verify from your RHEL VM that it can reach the Windows exporter:

   curl http://192.168.0.105:9182/metrics | head -20


===============================
STEP 4 - INSTALL GRAFANA
===============================

1. Add Grafana repository:

   Create file /etc/yum.repos.d/grafana.repo with content:

   [grafana]
   name=grafana
   baseurl=https://packages.grafana.com/oss/rpm
   repo_gpgcheck=1
   enabled=1
   gpgcheck=1
   gpgkey=https://packages.grafana.com/gpg.key

2. Install and start:

   sudo dnf install grafana -y
   sudo systemctl enable --now grafana-server

3. Access Grafana in your browser:
   http://192.168.8.100:3000
   Default login: admin / admin

4. Add Prometheus as data source:
   - Go to Connections > Data Sources > Add data source
   - Select Prometheus
   - Set URL to: http://localhost:9091
   - Click Save & Test


===============================
STEP 5 - IMPORT DASHBOARDS IN GRAFANA
===============================

Go to Dashboards > Import and use these Grafana dashboard IDs:

   Node Exporter Full    --> ID: 1860
   Windows Exporter      --> ID: 14694

After import, select the correct JOB from the dropdown at the top
of the dashboard:
   - For Linux panels  : select node_exporter
   - For Windows panels: select windows_exporter


===============================
TROUBLESHOOTING
===============================

--- Check if scrape targets are UP ---

   curl -s http://localhost:9091/api/v1/targets | python3 -m json.tool | grep -E "job|health|lastError"

--- Verify a metric exists ---

   curl -s 'http://localhost:9091/api/v1/query?query=windows_os_info' | python3 -m json.tool

--- List all windows metrics ---

   curl -s 'http://localhost:9091/api/v1/label/__name__/values' | python3 -m json.tool | grep windows


--- METRIC NAME MISMATCHES ---

Newer versions of windows_exporter use different metric names than
older Grafana dashboard templates expect. Known replacements:

   OLD NAME (dashboard expects)           NEW NAME (exporter provides)
   ----------------------------           ----------------------------
   windows_cs_physical_memory_bytes   --> windows_memory_physical_total_bytes
   windows_os_physical_memory_free_bytes --> windows_memory_physical_free_bytes
   windows_os_process_count           --> windows_system_processes
   windows_cs_logical_processors      --> windows_cpu_logical_processor

To fix: click the panel title > Edit > Code tab > replace old metric name with new one.


--- USEFUL PROMQL FIXES ---

Memory usage percentage:
   100.0 - 100 * windows_memory_physical_free_bytes{job=~"$job"} / windows_memory_physical_total_bytes{job=~"$job"}

Boot / Start-up time:
   windows_system_boot_time_timestamp{instance="$instance"} * 1000
   (Set panel unit to "Datetime local" in Field settings)

CPU usage:
   100 - (avg by (instance) (rate(windows_cpu_time_total{mode="idle",instance="$instance"}[5m])) * 100)


===============================
ENVIRONMENT DETAILS
===============================

Monitoring VM  : RHEL 10 (rhel10-srv)
               : VMware Workstation

Windows Host   : Windows 11 Home
               : windows_exporter on port 9182

Prometheus     : Port 9091 (non-default, conflict with websm on 9090)
Grafana        : Port 3000
Node Exporter  : Port 9100
