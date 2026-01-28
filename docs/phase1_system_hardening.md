## Phase 1: System Hardening & Base Configuration

### Objective

Secure a fresh Raspberry Pi OS install and prepare it for production services.

### Core Tasks

* User & SSH hardening
* Static IP configuration
* Firewall setup
* Service minimization
* System updates

---

### Terminal Compatibility Fix (kitty)

**Issue:**

```
'xterm-kitty': unknown terminal type
```

**Fix (client-side):**
Added to kitty config:

```
term xterm-256color
```

**Concept Learned:**

* `$TERM` must be supported on the remote system

Reference:

* [https://sw.kovidgoyal.net/kitty/faq/](https://sw.kovidgoyal.net/kitty/faq/)

---

### Package Management

Installed common admin tools:

```bash
sudo apt update
sudo apt upgrade
sudo apt install vim ufw
```

---

### SSH Hardening

**Key Concepts:**

* `ssh` = client
* `sshd` = server daemon

**Steps:**

```bash
ssh-keygen
ssh-copy-id user@host
```

On server:

```bash
sudo vim /etc/ssh/sshd_config
```

Set:

```
PasswordAuthentication no
```

Verify:

```bash
sshd -T | grep passwordauthentication
```

Restart service:

```bash
sudo systemctl restart sshd
```

---

### Firewall Setup (UFW)

```bash
sudo ufw allow ssh/tcp
sudo ufw logging on
sudo ufw enable
sudo ufw status
```

**Concepts Learned:**

* Always allow SSH before enabling firewall
* Logging helps with later intrusion detection

Reference:

* [https://wiki.ubuntu.com/UncomplicatedFirewall](https://wiki.ubuntu.com/UncomplicatedFirewall)

---

### Disabling Unnecessary Services

Before disabling services, checked reverse dependencies:

```bash
systemctl list-dependencies SERVICE --reverse
```

Disabled services:

* **Bluetooth**

```bash
systemctl disable --now bluetooth
```

* **CUPS (printing)**

```bash
systemctl disable --now cups
```

* **LightDM (display manager)**

```bash
systemctl disable --now lightdm
```

* **ModemManager**

```bash
systemctl disable --now ModemManager
```

* **NFS utilities**

```bash
systemctl disable --now nfs-blkmap rpcbind
```

* **colord** (static daemon)

```bash
systemctl mask colord
```

* **accounts-daemon** (GUI-related)

```bash
systemctl disable --now accounts-daemon
```

**Concepts Learned:**

* Difference between `stop`, `disable`, and `mask`
* Static vs triggered daemons

---

### Avahi (mDNS) on Client Machine

Enabled service discovery on Arch Linux client:

```bash
sudo pacman -S avahi nss-mdns
sudo systemctl enable --now avahi-daemon
```

Edited `/etc/nsswitch.conf`:

```
hosts: files mdns_minimal [NOTFOUND=return] dns
```

---

### Static IP Configuration (NetworkManager)

**Commands Used:**

```bash
hostname -I
ip r | grep default
nmcli connection show
```

**Static IP Assignment (safe placeholders):**

```bash
sudo nmcli connection modify "MyWiFiNetwork" \
ipv4.addresses 192.168.x.200/24 \
ipv4.gateway 192.168.x.1 \
ipv4.dns 192.168.x.1 \
ipv4.method manual
```

Apply:

```bash
sudo nmcli connection down "MyWiFiNetwork"
sudo nmcli connection up "MyWiFiNetwork"
```

Verify:

```bash
hostname -I
```

**Important Gotcha:**

* Running `nmcli connection down` over **Wi-Fi + SSH will drop your session**
* Use Ethernet or local console to avoid lockout

---

## Key Concepts Learned

* Daemons are background services managed by systemd
* Always check reverse dependencies before disabling services
* Static IPs must:

  * Stay within subnet
  * Use router as gateway
  * Avoid DHCP pool conflicts
* SSH keys are safer than passwords

---

## Sysadmin Skills Practiced

* Linux service management (systemd)
* Network configuration & routing
* Firewall rule design
* SSH security & recovery
* Dependency analysis
* Documentation & incident recovery

---

## References

* [https://man7.org/linux/man-pages/man1/systemctl.1.html](https://man7.org/linux/man-pages/man1/systemctl.1.html)
* [https://www.freedesktop.org/wiki/Software/AccountsService/](https://www.freedesktop.org/wiki/Software/AccountsService/)
* [https://wiki.archlinux.org/title/Avahi](https://wiki.archlinux.org/title/Avahi)
* [https://askubuntu.com/questions/1440290/unable-to-disable-password-authentication-over-ssh](https://askubuntu.com/questions/1440290/unable-to-disable-password-authentication-over-ssh)
* [https://www.instructables.com/How-to-Set-a-Static-IP-on-Raspberry-Pi-The-Right-W/](https://www.instructables.com/How-to-Set-a-Static-IP-on-Raspberry-Pi-The-Right-W/)
