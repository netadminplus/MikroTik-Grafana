# MikroTik Router Monitoring with Grafana

> **A complete Docker-based monitoring solution for MikroTik RouterOS v7 devices using Grafana, Prometheus, and SNMP Exporter.**

*Created by [Ramtin Rahmani Nejad](https://netadminhub.com) | [netadminhub](https://github.com/netadminhub)*

## Architecture

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   MikroTik      │ SNMP │   SNMP          │      │   Grafana       │
│   RouterOS v7   │─────▶│   Exporter      │─────▶│   Dashboard     │
│                 │      │   (Port 9116)   │      │   (Port 3000)   │
└─────────────────┘      └────────┬────────┘      └────────┬────────┘
                                 │                        │
                                 │ Prometheus Metrics     │
                                 ▼                        │
                          ┌─────────────────┐             │
                          │   Prometheus    │◀────────────┘
                          │   (Port 9090)   │
                          └─────────────────┘
```

## Prerequisites

- Ubuntu Server (20.04 or later recommended)
- MikroTik RouterOS v7 device
- Network connectivity between the monitoring server and MikroTik router

## Installation

### 1. Install Docker and Docker Compose

Follow the official Docker installation guide for Ubuntu:
- **Docker**: https://docs.docker.com/engine/install/ubuntu/
- **Docker Compose**: https://docs.docker.com/compose/install/

Quick installation (official Docker script):
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Add your user to the docker group (optional, to run docker without sudo):
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Clone or Download This Repository

```bash
cd /opt
git clone <repository-url> mikrotik-monitoring
cd mikrotik-monitoring
```

Or download as ZIP and extract:
```bash
wget <zip-url> -O mikrotik-monitoring.zip
unzip mikrotik-monitoring.zip
cd mikrotik-monitoring
```

### 3. Configure Environment Variables

Copy the example environment file and customize credentials:

```bash
cp .env.example .env
nano .env
```

**Recommended**: Change the default password in `.env`:
```bash
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=your-secure-password
```

### 4. Configure MikroTik Router

Connect to your MikroTik router via WinBox, WebFig, or SSH and run the following commands:

#### Enable SNMP Server

```routeros
# Enable SNMP service
/snmp set enabled=yes

# Set SNMP community (change 'public' to your preferred community string)
/snmp community add name=public address=0.0.0.0/0 security=authorized read-access=yes

# Set SNMP contact and location (optional but recommended)
/system identity set name="MyMikroTik"
/snmp set contact="admin@example.com" location="Server Room"
```

#### Configure SNMP Access (More Secure)

For better security, restrict SNMP access to your monitoring server's IP:

```routeros
# Remove the default open community
/snmp community remove [find where name="public"]

# Add community restricted to monitoring server IP only
/snmp community add name=public address=192.168.88.100/32 security=authorized read-access=yes
```

Replace `192.168.88.100` with your Ubuntu server's IP address.

#### Verify SNMP Configuration

Test SNMP from your Ubuntu server (choose one method):

**Option A: Install SNMP tools on host (optional)**
```bash
sudo apt update
sudo apt install -y snmp snmp-mibs-downloader

# Test SNMP connection (replace with your router IP)
snmpwalk -v2c -c public 192.168.88.1 system
```

**Option B: Test via Docker (no host installation required)**
```bash
docker run --rm -it --network host alpine sh -c "apk add --no-cache net-snmp-tools && snmpwalk -v2c -c public 192.168.88.1 system"
```

You should see system information returned.

### 5. Update Prometheus Configuration

Edit the Prometheus configuration to match your MikroTik router's IP:

```bash
nano prometheus/prometheus.yml
```

Find and update the target IP address:
```yaml
- job_name: 'mikrotik'
  static_configs:
    - targets:
        - 192.168.88.1  # <-- Change this to your router's IP
```

If you changed the SNMP community string, update it in `snmp_exporter/snmp.yml`:
```yaml
auths:
  public_v2:
    community: your-community-string  # <-- Change this
```

### 6. Start the Monitoring Stack

```bash
docker compose up -d
```

### 7. Verify Services Are Running

```bash
docker compose ps
```

All services should show "healthy" status.

Check logs if needed:
```bash
docker compose logs -f
```

### 8. Access Grafana

Open your web browser and navigate to:

```
http://<your-server-ip>:3000
```

Default credentials:
- **Username**: `admin`
- **Password**: `admin` (or what you set in `.env`)

You will be prompted to change the password on first login.

### 9. View the Dashboard

After logging in:

1. Click on **Dashboards** in the left sidebar
2. Select **MikroTik Router Dashboard**
3. Use the "Router" dropdown at the top to select your device

## Dashboard Panels

The dashboard includes:

### System Overview
- CPU Load (gauge + history)
- Memory Usage (gauge + history)
- Disk Usage (gauge)
- Temperature (gauge, if supported by device)

### Network Interfaces
- Interface Traffic (inbound/outbound)
- Interface Status table
- Interface Packets (inbound/outbound)
- Interface Errors and Discards

### Connection Tracking & Firewall
- Connection Tracking count
- TCP Connections
- TCP Connection Opens

### System Info
- Router Name
- System Description
- Uptime
- Location
- Contact
- CPU Frequency

## Managing the Stack

### Start
```bash
docker compose up -d
```

### Stop
```bash
docker compose down
```

### Restart
```bash
docker compose restart
```

### View Logs
```bash
docker compose logs -f
docker compose logs -f prometheus
docker compose logs -f grafana
docker compose logs -f snmp_exporter
```

### Update
```bash
docker compose pull
docker compose up -d --force-recreate
```

### Backup Data

Grafana data (dashboards, users, settings):
```bash
tar -czf grafana-backup-$(date +%Y%m%d).tar.gz grafana/data/
```

Prometheus data (metrics history):
```bash
tar -czf prometheus-backup-$(date +%Y%m%d).tar.gz prometheus/data/
```

## Troubleshooting

### SNMP Connection Issues

1. **Verify SNMP is enabled on MikroTik:**
   ```routeros
   /snmp print
   /snmp community print
   ```

2. **Check firewall rules on MikroTik:**
   ```routeros
   /ip firewall filter print
   # Ensure UDP port 161 is allowed from monitoring server
   ```

3. **Test from Ubuntu server:**
   ```bash
   snmpwalk -v2c -c public 192.168.88.1 system
   ```

4. **Check SNMP exporter logs:**
   ```bash
   docker compose logs snmp_exporter
   ```

### No Data in Grafana

1. **Verify Prometheus is scraping:**
   - Open http://<server-ip>:9090/targets
   - Check if "mikrotik" job is UP

2. **Query Prometheus directly:**
   - Go to http://<server-ip>:9090
   - Try query: `mtxrCpuLoad`

3. **Check time range:**
   - Ensure dashboard time range includes current time

### Container Issues

1. **Check container status:**
   ```bash
   docker compose ps -a
   ```

2. **Restart specific service:**
   ```bash
   docker compose restart prometheus
   ```

3. **Rebuild containers:**
   ```bash
   docker compose down
   docker compose up -d --build
   ```

## Adding Multiple Routers

To monitor multiple MikroTik routers:

1. Edit `prometheus/prometheus.yml`:
   ```yaml
   - job_name: 'mikrotik'
     static_configs:
       - targets:
           - 192.168.88.1
           - 192.168.88.2
           - 192.168.89.1
     metrics_path: /snmp
     params:
       auth: [public_v2]
       module: [mikrotik]
     relabel_configs:
       - source_labels: [__address__]
         target_label: __param_target
       - source_labels: [__param_target]
         target_label: instance
       - target_label: __address__
         replacement: snmp_exporter:9116
   ```

2. Reload Prometheus config:
   ```bash
   curl -X POST http://localhost:9090/-/reload
   ```

3. The dashboard will automatically show all routers in the dropdown.

## Security Considerations

1. **Change default credentials** in `.env` file
2. **Restrict SNMP access** to monitoring server IP only
3. **Use a strong SNMP community string** (not "public")
4. **Enable firewall rules** to restrict access to ports 3000, 9090, 9116
5. **Consider HTTPS** for Grafana in production (use reverse proxy)
6. **Regular backups** of Grafana data

## Customization

### Modify Scrape Interval

Edit `prometheus/prometheus.yml`:
```yaml
global:
  scrape_interval: 30s  # Change as needed
```

### Change Data Retention

Edit `docker-compose.yml` under prometheus command:
```yaml
- '--storage.tsdb.retention.time=15d'  # Change retention period
```

### Add Custom Dashboards

1. Create JSON dashboard file in `grafana/dashboards/`
2. Dashboard will auto-load within 30 seconds
3. Or import via Grafana UI

## Ports Used

| Service     | Port | Description                    |
|-------------|------|--------------------------------|
| Grafana     | 3000 | Web UI                         |
| Prometheus  | 9090 | Metrics & query interface      |
| SNMP Exporter | 9116 | SNMP to Prometheus translator |
| MikroTik    | 161  | SNMP agent (UDP)               |

## File Structure

```
mikrotik-monitoring/
├── docker-compose.yml          # Main Docker Compose configuration
├── .env.example                # Example environment variables
├── prometheus/
│   ├── prometheus.yml          # Prometheus configuration
│   └── data/                   # Prometheus data storage
├── snmp_exporter/
│   └── snmp.yml                # SNMP exporter configuration
├── grafana/
│   ├── data/                   # Grafana data storage
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasources.yml # Datasource configuration
│   │   └── dashboards/
│   │       └── dashboards.yml  # Dashboard provisioning
│   └── dashboards/
│       └── mikrotik-dashboard.json  # Main dashboard
└── README.md                   # This file
```

## Support

For issues related to:
- **Grafana**: https://grafana.com/docs/
- **Prometheus**: https://prometheus.io/docs/
- **SNMP Exporter**: https://github.com/prometheus/snmp_exporter
- **MikroTik**: https://help.mikrotik.com/

## Author

**Built between pings by [Ramtin Rahmani Nejad](https://netadminhub.com) ❤️**

🌐 **Website**: [netadminhub.com](https://netadminhub.com)  
💻 **GitHub**: [github.com/netadminhub](https://github.com/netadminhub)  
📺 **YouTube**: [youtube.com/@netadminhub](https://youtube.com/@netadminhub)  
📸 **Instagram**: [instagram.com/netadminhub](https://instagram.com/netadminhub)  
✕ **X (Twitter)**: [x.com/netadminhub](https://x.com/netadminhub)  
✈️ **Telegram**: [t.me/netadminhub](https://t.me/netadminhub)

### Support This Project

If you find this project useful, consider supporting to keep the content free and independent:

🍕 **Donate**: [netadminhub.com/donate/](https://netadminhub.com/donate/)

## License

This configuration is provided as-is for monitoring MikroTik devices.
