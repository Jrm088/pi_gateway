## Phase 5: System Monitoring (Netdata + Uptime Kuma)

### Objective

Install and configure Netdata for real-time system metrics monitoring and Uptime Kuma for service availability monitoring, providing comprehensive visibility into the Pi's performance and service health.

### Prerequisites

* Completed Phase 4 (security services configured)
* SSH access to the Raspberry Pi
* Basic understanding of Docker and containerization
* Firewall configured (UFW)

---

### Step 1: Install Netdata

**Update system:**

```bash
sudo apt update && sudo apt upgrade
```

**Download and run Netdata installation script:**

The version of Netdata in the Raspberry Pi OS repository is outdated and not managed by the official Netdata team. Use the official installation script instead:

```bash
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh
```

---

### Step 2: Configure Netdata Web Interface Access

**Verify Netdata is listening on port 19999:**

```bash
ss -tulpn | grep 19999
```

**Expected output:**

```
tcp   LISTEN 0      4096         0.0.0.0:19999      0.0.0.0:*    users:(("netdata",pid=23227,fd=6))
tcp   LISTEN 0      4096            [::]:19999         [::]:*    users:(("netdata",pid=23227,fd=8))
```

**Check UFW firewall status:**

```bash
sudo ufw status
```

If port 19999 is not listed, you need to allow it through the firewall.

**Allow Netdata through UFW:**

```bash
sudo ufw allow 19999/tcp
sudo systemctl reload ufw
```

**Access Netdata web interface:**

Open a browser and navigate to:

```
http://192.168.1.100:19999
```

Replace `192.168.1.100` with your Pi's IP address.

---

### Step 3: Install Docker

Docker is required to run Uptime Kuma in a container.

**Install Docker using official script:**

```bash
curl -sSL https://get.docker.com | sh
```

**Add your user to the Docker group:**

Docker is a service that runs in the background and requires root-level privileges. If your user isn't root and isn't in the docker group, it can't send commands to the Docker daemon.

```bash
sudo usermod -aG docker $USER
```

Replace `$USER` with your username if needed.

**Reboot to apply group changes:**

```bash
sudo reboot
```

**Verify Docker group membership:**

```bash
groups
```

You should see `docker` listed in the output.

**Test Docker installation:**

```bash
docker run hello-world
```

**Check Docker version:**

```bash
docker --version
```

**Expected output:**

```
Docker version 29.1.5, build 0e6fee6
```

---

### Step 4: Install Uptime Kuma

**Create directory for Uptime Kuma:**

```bash
sudo mkdir -p /opt/stacks/uptimekuma
cd /opt/stacks/uptimekuma
```

**Create Docker Compose file:**

```bash
sudo vim compose.yaml
```

**Add the following configuration:**

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    volumes:
      - ./data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - 3001:3001
    restart: unless-stopped
```

**Start Uptime Kuma container:**

```bash
docker compose up -d
```

The `-d` flag runs the service in the background (detached mode).

**Get your Pi's IP address:**

```bash
hostname -I
```

**Access Uptime Kuma web interface:**

Open a browser and navigate to:

```
http://192.168.1.100:3001
```

Replace `192.168.1.100` with your Pi's IP address.

**Create login credentials** on first access.

---

### Step 5: Configure Uptime Kuma Monitors

#### Monitor Pi-Server (Ping)

**Add new monitor:**

* Click "Add New Monitor"
* **Monitor Type:** Ping
* **Friendly Name:** pi-server
* **Hostname:** 192.168.1.100
* Click "Save"

#### Monitor Second Server (Ping)

**Add new monitor:**

* Click "Add New Monitor"
* **Monitor Type:** Ping
* **Friendly Name:** server-2
* **Hostname:** 192.168.1.102
* Click "Save"

#### Monitor Pi-Server SSH (TCP Port)

**Add new monitor:**

* Click "Add New Monitor"
* **Monitor Type:** TCP Port
* **Friendly Name:** pi-server ssh
* **Hostname:** 192.168.1.100
* **Port:** 22
* Click "Save"

#### Monitor Pi-hole Dashboard (HTTP)

**Add new monitor:**

* Click "Add New Monitor"
* **Monitor Type:** HTTP(s)
* **Friendly Name:** pi hole dashboard
* **URL:** http://192.168.1.100/admin/login
* Click "Save"

#### Monitor Netdata Web Interface (HTTP)

**Add new monitor:**

* Click "Add New Monitor"
* **Monitor Type:** HTTP(s)
* **Friendly Name:** netdata web interface
* **URL:** http://192.168.1.100:19999
* Click "Save"

**Note:** Replace IP addresses with your actual network configuration.

---

### How It Works

#### Netdata: Real-Time System Monitoring

Netdata is an open-source tool that collects real-time metrics for system performance, including CPU usage, memory consumption, disk activity, network traffic, and more.

**Key Features:**

* **Real-time monitoring:** Metrics updated every second
* **Zero configuration:** Works out of the box with automatic detection
* **Low resource usage:** Minimal impact on system performance
* **Web interface:** Easy-to-read dashboards accessible via browser
* **Extensive metrics:** Monitors CPU, RAM, disk I/O, network, processes, and more

**How Netdata Works:**

1. Netdata collects metrics from the system at 1-second intervals
2. Data is stored in memory for recent history (default: 1 hour)
3. Web interface displays real-time graphs and statistics
4. No external database required - all data stored locally

#### Uptime Kuma: Service Availability Monitoring

Uptime Kuma is a self-hosted monitoring tool that tracks the availability and response time of services and servers.

**Key Features:**

* **Multiple monitor types:** Ping, HTTP(s), TCP Port, DNS, and more
* **Status pages:** Create public status pages for your services
* **Notifications:** Alert via multiple channels (email, Slack, Discord, etc.)
* **Docker-based:** Easy deployment and updates
* **Lightweight:** Minimal resource consumption

**How Uptime Kuma Works:**

1. Monitors check configured services at regular intervals
2. Records uptime percentage and response times
3. Sends notifications when services go down or recover
4. Displays historical uptime data and incident logs

#### Monitoring Strategy

**Netdata provides:**

* System resource metrics (CPU, RAM, disk, network)
* Process-level monitoring
* Performance analysis and troubleshooting

**Uptime Kuma provides:**

* Service availability tracking
* Downtime alerts
* Uptime statistics for critical services

Together, they offer comprehensive monitoring of both system health and service availability.

---

### Troubleshooting

#### Netdata Web Interface Not Accessible

**Issue:** Cannot access Netdata web interface at `http://IP:19999`

**Solution:**

```bash
# Verify Netdata is listening
ss -tulpn | grep 19999

# Check if port is allowed through firewall
sudo ufw status

# Allow port if needed
sudo ufw allow 19999/tcp
sudo systemctl reload ufw
```

#### Netdata Not Showing Additional Charts

**Issue:** After enabling charts in `charts.d.conf`, additional data isn't displayed.

**Reason:** Some features are not supported on aarch64 (ARM64) architecture, including:

* Processor information details
* CPU temperature monitoring with lm-sensors

**Note:** This is an architectural limitation, not a configuration issue.

#### Docker Permission Denied

**Issue:** Cannot run Docker commands without sudo.

**Solution:**

```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Reboot to apply changes
sudo reboot

# Verify group membership
groups
```

#### Uptime Kuma Monitors Not Working

**Issue:** All monitors show as down or unreachable.

**Common causes:**

1. **Incorrect IP addresses or hostnames** - Double-check all configured addresses
2. **Wrong monitor type** - Ensure the monitor type matches the service (HTTP for web, TCP for ports, Ping for connectivity)
3. **Firewall blocking** - Verify required ports are open
4. **Typos in configuration** - Review each monitor's settings carefully

**Solution:**

* Edit each monitor in Uptime Kuma
* Verify IP addresses, ports, and URLs are correct
* Change monitor type if necessary
* Test connectivity manually from the Pi

**Example troubleshooting:**

```bash
# Test ping
ping 192.168.1.100

# Test TCP port
nc -zv 192.168.1.100 22

# Test HTTP
curl -I http://192.168.1.100:19999
```

---

### Verification Checklist

* Netdata service is running
* Netdata web interface accessible on port 19999
* Port 19999 allowed through UFW
* Docker installed and user added to docker group
* Docker running without sudo
* Uptime Kuma container running
* Uptime Kuma web interface accessible on port 3001
* All monitors configured in Uptime Kuma
* All monitors showing "Up" status (green)
* Monitor response times displaying correctly

---

### Sysadmin Skills Practiced

* System monitoring tool installation (Netdata)
* Docker installation and configuration
* Docker user permissions management
* Docker Compose file creation
* Container deployment and management
* Service availability monitoring (Uptime Kuma)
* Firewall port management
* Web-based dashboard configuration
* Network troubleshooting
* Monitor configuration for various service types

---

### References

* [Install Netdata on Raspberry Pi](https://pimylifeup.com/raspberry-pi-netdata/)
* [Netdata ARM64 Processor Info Issue](https://github.com/netdata/netdata/issues/12104)
* [Netdata Charts.d Plugin Documentation](https://learn.netdata.cloud/docs/developer-and-contributor-corner/external-plugins/charts.d.plugin)
* [lm-sensors ARM Architecture Limitations](https://github.com/lm-sensors/lm-sensors/issues/30)
* [Enable Netdata Temperature Monitoring](https://www.chienlit.com/how-to-enable-netdata-temperature-monitoring-in-a-second/)
* [Install Docker on Raspberry Pi](https://pimylifeup.com/raspberry-pi-docker/)
* [Monitor Services with Uptime Kuma](https://serverstadium.com/knowledge-base/monitor-your-services-uptime-using-uptime-kuma/)

---

## Project Complete! 🎉

All phases have been successfully implemented and documented. This Raspberry Pi Security Gateway now provides:

✅ Network-wide ad blocking (Pi-hole)  
✅ VPN routing for all devices (ProtonVPN + WireGuard)  
✅ Intrusion prevention (Fail2Ban + CrowdSec)  
✅ Real-time monitoring (Netdata + Uptime Kuma)  
✅ Comprehensive firewall management (UFW/iptables)

### Next Steps

* Fine-tune monitoring alerts and thresholds
* Consider implementing automated backups
* Explore additional CrowdSec scenarios for enhanced protection
* Document any additional customizations or optimizations

---

## License

This project documentation is provided as-is for educational purposes. Feel free to use and modify for your own homelab projects.

## Acknowledgments

* Imran Afzal for the excellent RHCSA Udemy course that started this journey
* The open-source communities behind Pi-hole, WireGuard, Fail2Ban, CrowdSec, Netdata, and Uptime Kuma
* All the tutorial authors and documentation contributors referenced throughout this project
