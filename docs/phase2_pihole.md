## Phase 2: Pi-hole Installation & Configuration

### Objective
Install Pi-hole for DNS-based ad blocking, configure initial settings, test functionality on a single client device (laptop), and troubleshoot installation issues.

---

## Installation

### Run Pi-hole Installer
```bash
curl -sSL https://install.pi-hole.net | bash
```

### Installer (TUI) Selections
* Static IP warning: **continue**
* Confirm static IP: **continue**
* Upstream DNS provider: **Cloudflare**
* Blocklists: **yes**
* Admin Web Interface: **yes**
* Query logging: **no**
* Privacy mode (FTL): **0 (show everything)**

---

## Post-Install Configuration

### Change Web Admin Password
```bash
pihole setpassword
```

### Access Web Interface
```
http://192.168.x.x:80/admin
```

### Blocklist Management
**Group Management → Adlists**

Added malware blocklist:
```
https://raw.githubusercontent.com/PolishFiltersTeam/KADhosts/master/KADhosts.txt
```

**Tools → Update Gravity** to apply changes

---

## Client-Side DNS Configuration (Laptop Only)

Configured DNS on laptop only to avoid affecting other network devices.

### Verify Current DNS Settings
```bash
nmcli con show
nmcli con show YourWiFiNetwork | grep dns
cat /etc/resolv.conf
```

### Set Pi-hole as DNS Server
```bash
sudo nmcli connection modify YourWiFiNetwork ipv4.dns "192.168.x.x"
```

### Verify DNS Assignment
```bash
nmcli con show YourWiFiNetwork | grep dns
```

---

## Issues Fixed & Troubleshooting

### Issue 1: Web Interface Not Detecting Laptop + Slow Internet

**Root Cause:** Firewall blocking DNS queries (port 53)

**Solution:** Open required Pi-hole ports
```bash
sudo ufw allow 53     # DNS (critical)
sudo ufw allow 80     # HTTP web interface
sudo ufw allow 81     # Custom lighttpd port
sudo ufw allow 443    # HTTPS (optional)
sudo ufw allow 4711   # Pi-hole FTL API (optional)
```

**Result:** Pi-hole dashboard now shows live queries, internet speed normal

---

### Issue 2: "Pi-hole's DNS Address Could Not Be Found"

**Investigation Steps:**

1. Verified Lighttpd was installed:
```bash
sudo apt install lighttpd
sudo systemctl enable --now lighttpd
```

2. Corrected Pi-hole interface binding:
```bash
sudo vim /etc/pihole/pihole.toml
```

Changed:
```ini
interface = "eth0"
```

To:
```ini
interface = "wlan0"
```

3. Restarted Pi-hole FTL:
```bash
sudo systemctl restart pihole-FTL
```

---

### Issue 3: Lighttpd Service Failures

**Problem:** Initial install failed due to incorrect interface selection (chose Ethernet instead of WiFi)

**Actions Taken:**
* Removed and reinstalled Pi-hole with correct interface
* Detected systemd degraded state:
```bash
systemctl reset-failed
```

---

### Issue 4: Port 80 Conflict

**Diagnostic Commands Used:**
```bash
lighttpd -t -f /etc/lighttpd/lighttpd.conf  # Test config syntax
ss -tulpn | grep :80                         # Check port usage
journalctl -u lighttpd --no-pager            # View full logs
```

**Findings:**
* Port 80 already in use by `pihole-FTL`
* Missing certificate file: `/etc/lighttpd/certs/server.pem`
* Both IPv4 and IPv6 bindings attempted

**Solution: Move Lighttpd to Port 81**
```bash
sudo vim /etc/lighttpd/lighttpd.conf
```
Changed listening port to 81

**Verification:**
```bash
ss -tulpn | grep :81
# Output showed lighttpd successfully listening on port 81
```

---

### Issue 5: Self-Signed Certificate Generation

Generated SSL certificate for HTTPS (optional):
```bash
openssl genrsa -des3 -out testing.key 2048
openssl req -new -key testing.key -out testing.csr
openssl x509 -req -days 365 -in testing.csr -signkey testing.key -out testing.crt
cat testing.key testing.crt > certificate.pem
```

Restarted Lighttpd:
```bash
systemctl restart lighttpd
```

---

### Issue 6: Firewall Configuration

**Problem:** Firewall initially only allowed SSH (port 22)

**Solution:**
```bash
sudo ufw status
sudo ufw allow 80
sudo ufw allow 81
sudo systemctl restart ufw
```

**Final Resolution:**

Web interface failed because I was using `https://` instead of `http://`

**Correct URL:**
```
http://192.168.x.x:80/admin
```

---

## Key Concepts Learned

### Technical Concepts
* **Port 53 is critical for DNS services** - blocking it breaks all DNS queries
* **Port conflicts** - pihole-FTL serves both DNS (53) and web interface (80)
* **IPv4 vs IPv6 binding** - services can listen on both simultaneously
* **Firewall troubleshooting** - always check firewall when services can't communicate

### Troubleshooting Skills
* Systematic approach: logs → ports → services → firewall
* Using `journalctl` with `--no-pager` to see full error messages
* Checking what's listening on ports with `ss -tulpn`
* Testing service configs before restarting

### Pi-hole Architecture
* pihole-FTL handles DNS queries and web dashboard
* Lighttpd serves the admin web interface
* Query logging can be toggled for privacy
* Gravity database stores blocklists

---

## Notes

**Pi-hole Access Points:**
* Primary: `http://192.168.x.x:80/admin`
* Alternative: `http://pi.hole/admin` (when using Pi-hole as DNS)

**Port Bindings:**
* IPv4: `0.0.0.0:80` (all interfaces)
* IPv6: `[::]:80` (all interfaces)

**Current Configuration:**
* Basic setup complete and stable
* Cloudflare upstream DNS working
* Single device (laptop) testing successful
* Ads being blocked effectively

**Future Enhancement:**
* Consider switching from Cloudflare to Unbound for recursive DNS (Phase 2.5)

---

## Sysadmin Skills Practiced
* DNS server configuration
* Web service management (lighttpd)
* Port conflict resolution
* SSL/TLS certificate generation
* Firewall rule management
* Log analysis with journalctl
* Service debugging
* Client DNS configuration

---

## References
* [Pi-hole + Unbound Tutorial](https://www.crosstalksolutions.com/the-worlds-greatest-pi-hole-and-unbound-tutorial-2023/)
* [Pi-hole Upstream DNS Discussion](https://discourse.pi-hole.net/t/how-do-i-choose-an-upstream-dns-server/258/10)
* [Systemd Degraded State](https://unix.stackexchange.com/questions/447561/systemctl-status-shows-state-degraded)
* [Lighttpd Documentation](https://redmine.lighttpd.net/projects/lighttpd/wiki/GetLighttpd)
* [Check Port Usage on Linux](https://www.cyberciti.biz/faq/find-linux-what-running-on-port-80-command/)
* [Lighttpd HTTPS Setup](https://stackoverflow.com/questions/19938022/lighttpd-https-installation-with-self-signed-certificate)
* [Check Open Ports on Raspberry Pi](https://linuxconfig.org/how-to-check-open-ports-on-raspberry-pi)
* [Pi-hole Blocklists](https://firebog.net/)
* [nmcli DNS Configuration](https://infotechys.com/change-dns-settings-using-the-nmcli-utility/)
* [Pi-hole Required Ports](https://discourse.pi-hole.net/t/what-ports-are-required-for-pihole-to-work/6203/6)
