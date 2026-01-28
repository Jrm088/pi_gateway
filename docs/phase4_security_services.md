## Phase 4: Security Services (Fail2Ban + CrowdSec)

### Objective

Install and configure Fail2Ban and CrowdSec to protect the Raspberry Pi from brute force attacks and malicious IPs, providing layered intrusion prevention with both reactive and proactive security measures.

### Prerequisites

* Completed Phase 3 (VPN gateway configured)
* SSH access to the Raspberry Pi
* Basic understanding of firewall rules and intrusion prevention

---

### Step 1: Fail2Ban Installation

**Update system and install Fail2Ban:**

```bash
sudo apt update && sudo apt upgrade
sudo apt install fail2ban
```

**Verify Fail2Ban is running:**

```bash
sudo systemctl status fail2ban
```

---

### Step 2: Configure Fail2Ban

**Important:** Do not edit `jail.conf` directly, as it will be overwritten by distribution updates. All customizations should be made in `jail.local` or `jail.d/customisation.local`.

**Create custom jail configuration:**

```bash
sudo -i
vim /etc/fail2ban/jail.local
```

**Add the following configuration:**

```ini
# enabled = enables jail

[DEFAULT]
bantime = 15m

[sshd]
enabled = true
mode   = extra
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

**Restart Fail2Ban to apply changes:**

```bash
systemctl restart fail2ban
```

**Check Fail2Ban logs:**

```bash
cat /var/log/fail2ban.log
```

**Expected output:**

```
2026-01-21 22:14:29,291 fail2ban.server         [14776]: INFO    Starting Fail2ban v1.1.0
2026-01-21 22:14:29,292 fail2ban.observer       [14776]: INFO    Observer start...
2026-01-21 22:14:29,301 fail2ban.database       [14776]: INFO    Connected to fail2ban persistent database
2026-01-21 22:14:29,303 fail2ban.jail           [14776]: INFO    Creating new jail 'sshd'
2026-01-21 22:14:29,312 fail2ban.jail           [14776]: INFO    Jail 'sshd' uses systemd {}
2026-01-21 22:14:29,313 fail2ban.jail           [14776]: INFO    Initiated 'systemd' backend
2026-01-21 22:14:29,349 fail2ban.filter         [14776]: INFO      maxRetry: 5
2026-01-21 22:14:29,349 fail2ban.filter         [14776]: INFO      findtime: 600
2026-01-21 22:14:29,349 fail2ban.actions        [14776]: INFO      banTime: 900
2026-01-21 22:14:29,351 fail2ban.jail           [14776]: INFO    Jail 'sshd' started
```

---

### Step 3: Test Fail2Ban

To verify Fail2Ban is working correctly, temporarily adjust the ban time for testing purposes.

**Temporarily change ban time:**

```bash
vim /etc/fail2ban/jail.local
```

Change `bantime` to:

```ini
bantime = 1m
```

**Restart Fail2Ban:**

```bash
systemctl restart fail2ban
```

**Test from another machine (VM or laptop):**

Attempt to SSH with an incorrect username 5 times:

```bash
ssh fakeuser@192.168.1.100
# Repeat 5 times with failed password attempts
```

**Check Fail2Ban logs on the Pi:**

```bash
cat /var/log/fail2ban.log
```

**Expected output showing the ban:**

```
2026-01-21 22:54:53,126 fail2ban.filter         [14942]: INFO    [sshd] Found 192.168.1.115
2026-01-21 22:54:55,843 fail2ban.filter         [14942]: INFO    [sshd] Found 192.168.1.115
2026-01-21 22:54:56,593 fail2ban.filter         [14942]: INFO    [sshd] Found 192.168.1.115
2026-01-21 22:54:57,342 fail2ban.filter         [14942]: INFO    [sshd] Found 192.168.1.115
2026-01-21 22:54:58,342 fail2ban.filter         [14942]: INFO    [sshd] Found 192.168.1.115
2026-01-21 22:54:58,377 fail2ban.actions        [14942]: NOTICE  [sshd] Ban 192.168.1.115
2026-01-21 22:55:58,535 fail2ban.actions        [14942]: NOTICE  [sshd] Unban 192.168.1.115
```

**Restore original ban time:**

After testing, change `bantime` back to your preferred setting (e.g., `15m`):

```bash
vim /etc/fail2ban/jail.local
systemctl restart fail2ban
```

---

### Step 4: CrowdSec Installation

**Update system:**

```bash
sudo apt update && sudo apt upgrade
```

**Install CrowdSec:**

```bash
curl https://install.crowdsec.net/ | sudo bash
sudo apt install crowdsec
```

**Install NFT-based firewall bouncer:**

**Important:** The CrowdSec agent must be installed before the firewall bouncer. If the bouncer is installed first, it won't be registered automatically with the agent.

```bash
sudo apt install crowdsec-firewall-bouncer-nftables
```

---

### Step 5: Verify CrowdSec Configuration

**List registered bouncers:**

```bash
sudo cscli bouncers list
```

**Expected output:**

```
────────────────────────────────────────────────────────────────────────
 Name                            IP Address  Valid  Last API pull
────────────────────────────────────────────────────────────────────────
 cs-firewall-bouncer-1769289462  127.0.0.1   ✔️     2026-01-24T21:23:04Z
────────────────────────────────────────────────────────────────────────
```

**Check active protection scenarios:**

```bash
sudo cscli scenarios list
```

**Expected output:**

```
───────────────────────────────────────────────────────────────────────
 Name                             📦 Status    Version  Local Path
───────────────────────────────────────────────────────────────────────
 crowdsecurity/ssh-bf             ✔️  enabled  0.3
 crowdsecurity/ssh-cve-2024-6387  ✔️  enabled  0.2
 crowdsecurity/ssh-generic-test   ✔️  enabled  0.2
 crowdsecurity/ssh-refused-conn   ✔️  enabled  0.1
 crowdsecurity/ssh-slow-bf        ✔️  enabled  0.4
 crowdsecurity/ssh-time-based-bf  ✔️  enabled  0.2
───────────────────────────────────────────────────────────────────────
```

**Reload CrowdSec to apply changes:**

```bash
sudo systemctl reload crowdsec
```

**Check CrowdSec status:**

```bash
sudo systemctl status crowdsec
```

**Verify UFW firewall rules:**

```bash
sudo ufw status numbered
```

**Example output:**

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 81                         ALLOW IN    Anywhere
[ 3] 80                         ALLOW IN    Anywhere
[ 4] 53                         ALLOW IN    Anywhere
[ 5] 443                        ALLOW IN    Anywhere
[ 6] 4711                       ALLOW IN    Anywhere
[ 7] 19999/tcp                  ALLOW IN    Anywhere
```

---

### How It Works

#### Fail2Ban: Reactive Security

Fail2Ban is an intrusion prevention software framework designed to protect against brute force attacks. It monitors log files for suspicious activity and takes action when malicious behavior is detected.

**Key Components:**

* **Jail:** The combination of a filter and an action
* **Filter:** Watches for specific malicious behavior patterns
* **Action:** What happens when the filter detects a threat (typically banning the IP)

**How Fail2Ban Works:**

1. Monitors system logs (e.g., `/var/log/auth.log` for SSH)
2. Detects failed authentication attempts
3. After a threshold is reached (default: 5 attempts in 10 minutes), the IP is banned
4. Adds firewall rules to block the offending IP for a specified time (default: 15 minutes)

#### CrowdSec: Proactive Security

CrowdSec is a collaborative security engine that shares threat intelligence across its network. It complements Fail2Ban by blocking known malicious IPs before they can attack your system.

**Key Components:**

* **Agent:** Analyzes logs and detects threats
* **Bouncer:** Enforces blocking decisions (firewall bouncer blocks IPs)
* **Community Blocklist:** Shares and receives threat intelligence from the CrowdSec network

**How CrowdSec Works:**

1. Monitors system logs for attack patterns
2. Detects malicious behavior using scenarios (detection rules)
3. Shares detected threats with the CrowdSec network
4. Receives and blocks IPs flagged by other CrowdSec users
5. Automatically blocks known malicious IPs before they can attempt an attack

#### Fail2Ban vs CrowdSec

| Feature | Fail2Ban | CrowdSec |
|---------|----------|----------|
| **Approach** | Reactive - blocks after detecting an attack | Proactive - blocks known threats before they attack |
| **Intelligence** | Local rules only | Crowd-sourced threat intelligence |
| **Scope** | Protects individual system | Benefits from global threat data |
| **Best for** | Stopping active attackers | Preventing known attackers from reaching your system |

**Together, they provide layered security:**

* **CrowdSec** blocks known malicious IPs proactively
* **Fail2Ban** blocks new attackers caught in the act

#### Default Protection

By default, CrowdSec protects SSH and scans system logs. The default scenarios cover common SSH attack patterns, which is sufficient for this setup since SSH is the primary remote access method.

---

### Troubleshooting

#### Fail2Ban Not Starting

**Check service status:**

```bash
sudo systemctl status fail2ban
sudo journalctl -u fail2ban -n 50
```

**Common issues:**

* Syntax errors in `jail.local`
* Missing log files
* Conflicting configurations

#### Fail2Ban Not Blocking IPs

**Verify jail is enabled:**

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

**Check if IPs are being detected:**

```bash
sudo tail -f /var/log/fail2ban.log
```

**Manually test the filter:**

```bash
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

#### CrowdSec Not Running

**Check service status:**

```bash
sudo systemctl status crowdsec
sudo journalctl -u crowdsec -n 50
```

**Verify bouncer is registered:**

```bash
sudo cscli bouncers list
```

**Check for blocked IPs:**

```bash
sudo cscli decisions list
```

#### Bouncer Not Installed Correctly

If the firewall bouncer was installed before the CrowdSec agent, it won't be registered automatically.

**Solution:**

```bash
# Remove and reinstall the bouncer
sudo apt remove crowdsec-firewall-bouncer-nftables
sudo apt install crowdsec-firewall-bouncer-nftables

# Verify registration
sudo cscli bouncers list
```

---

### Verification Checklist

* Fail2Ban service is active and enabled
* Fail2Ban jail for SSH is configured and running
* Fail2Ban successfully bans test IPs
* CrowdSec agent is installed and running
* CrowdSec firewall bouncer is installed and registered
* CrowdSec scenarios are active (SSH protection enabled)
* UFW firewall rules include necessary ports
* All services start automatically on boot

---

### Sysadmin Skills Practiced

* Intrusion prevention system (IPS) configuration
* Log monitoring and analysis
* Fail2Ban jail configuration
* CrowdSec agent and bouncer setup
* Threat intelligence integration
* Security testing and verification
* Service management with systemd
* Firewall rule management (UFW/nftables)

---

### References

* [Fail2Ban Wikipedia](https://en.wikipedia.org/wiki/Fail2ban)
* [Fail2Ban GitHub Repository](https://github.com/fail2ban/fail2ban)
* [Install Fail2Ban on Raspberry Pi](https://raspberrytips.com/install-fail2ban-raspberry-pi/)
* [How Fail2Ban Works - DigitalOcean](https://www.digitalocean.com/community/tutorials/how-fail2ban-works-to-protect-services-on-a-linux-server)
* [Using Fail2Ban to Secure Your Server](https://www.plesk.com/blog/various/using-fail2ban-to-secure-your-server/)
* [Configure Fail2Ban for Common Services](https://webdock.io/en/docs/how-guides/security-guides/how-configure-fail2ban-common-services)
* [Fail2Ban - Arch Linux Wiki](https://wiki.archlinux.org/title/Fail2ban)
* [Fail2Ban SSH Filter Discussion](https://forums.raspberrypi.com/viewtopic.php?t=234907)
* [Understanding Fail2Ban SSH Filter](https://stackoverflow.com/questions/59996070/how-to-understand-if-the-fail2ban-ssh-filter-is-working-with-a-new-port)
* [Secure Raspberry Pi with CrowdSec](https://www.crowdsec.net/blog/how-to-secure-your-raspberry-pi-with-crowdsec)
* [CrowdSec Firewall Bouncer Documentation](https://docs.crowdsec.net/u/bouncers/firewall/)
* [Protect Raspberry Pi with CrowdSec](https://www.polimetro.com/en/How-to-protect-your-Raspberry-Pi-with-CrowdSec)
