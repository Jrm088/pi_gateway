# Phase 3: VPN Gateway Configuration

## Objective
Configure the Raspberry Pi as a VPN gateway using ProtonVPN and WireGuard, allowing all network traffic to route through an encrypted VPN tunnel with proper NAT configuration.

## Prerequisites
- Completed Phase 2 (Pi-hole installed and working)
- Active ProtonVPN account
- Understanding of basic networking concepts (IP routing, gateways, NAT)

---

## Step 1: WireGuard Installation

**Install WireGuard and tools:**
```bash
sudo apt-get install wireguard wireguard-tools
```

**Initial configuration:**
```bash
sudo -i
cd /etc/wireguard
umask 077
```

**Generate encryption keys:**
```bash
wg genkey | tee server.key | wg pubkey > server.pub
```

**Create initial WireGuard configuration:**
```bash
vim /etc/wireguard/wg0.conf
```

Add:
```ini
[Interface]
Address = 10.100.0.1/24, fd08:4711::1/64
ListenPort = 47111
```

**Add private key to configuration:**
```bash
echo "PrivateKey = $(cat server.key)" >> /etc/wireguard/wg0.conf
exit
```

**Enable and start WireGuard service:**
```bash
sudo systemctl enable wg-quick@wg0.service
sudo systemctl daemon-reload
sudo systemctl start wg-quick@wg0
```

**Verify WireGuard is running:**
```bash
wg
```

**Configure Pi-hole for local-only DNS:**
- Access Pi-hole web interface
- Navigate to Settings → DNS
- Select "Allow only local requests"

---

## Step 2: ProtonVPN Configuration

**Download ProtonVPN WireGuard configuration:**
1. Log in to https://account.protonvpn.com/login
2. Click "Downloads" in the left sidebar
3. Select device name
4. Choose platform: Linux
5. Select VPN protocol: WireGuard
6. Choose a server location
7. Download the configuration file

**Install required dependency:**
```bash
sudo apt install resolvconf
```

**Replace WireGuard configuration with ProtonVPN config:**
- Copy the downloaded ProtonVPN configuration to `/etc/wireguard/wg0.conf`

**Restart WireGuard with new configuration:**
```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```

**Verify VPN connection:**
```bash
sudo wg show
curl ifconfig.me
```

The IP address should show a ProtonVPN server IP, not your home ISP IP.

**Ensure WireGuard starts on boot:**
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl daemon-reload
sudo systemctl start wg-quick@wg0
```

---

## Step 3: Enable IP Forwarding

IP forwarding allows the Pi to route traffic between network interfaces (acting as a gateway).

**Check current IPv4 forwarding status:**
```bash
sysctl net.ipv4.ip_forward
```
- `0` = disabled
- `1` = enabled

**Enable IPv4 forwarding permanently:**
```bash
sudo vim /etc/sysctl.d/99-ip-forwarding.conf
```

Add:
```ini
# Enable IPv4 forwarding
net.ipv4.ip_forward=1

# Disable IPv6 to prevent VPN leaks
net.ipv6.conf.all.disable_ipv6=1
```

**Apply changes:**
```bash
sudo sysctl -p /etc/sysctl.d/99-ip-forwarding.conf
```

Or reboot:
```bash
sudo reboot
```

**Verify settings:**
```bash
sysctl net.ipv4.ip_forward          # Should show: 1
sysctl net.ipv6.conf.all.disable_ipv6  # Should show: 1
```

### Troubleshooting: WireGuard IPv6 Error

**Issue:** WireGuard service fails with "IPv6 is disabled on nexthop device"

**Solution:**
```bash
sudo vim /etc/wireguard/wg0.conf
```

Remove any IPv6 addresses from `AllowedIPs` (entries containing `::/0`)

**Restart service:**
```bash
sudo systemctl restart wg-quick@wg0
sudo systemctl status wg-quick@wg0
```

---

## Step 4: NAT Configuration (MASQUERADE)

Network Address Translation (NAT) allows multiple devices to share the VPN's public IP address.

**Check current FORWARD chain policy:**
```bash
sudo iptables -L FORWARD -v
```

If the policy shows `DROP`, traffic forwarding is blocked by default.

### Configure UFW for Forwarding

**Enable forwarding policy:**
```bash
sudo vim /etc/default/ufw
```

Change:
```
DEFAULT_FORWARD_POLICY="DROP"
```

To:
```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

### Add MASQUERADE Rules

**Edit UFW's pre-routing rules:**
```bash
sudo vim /etc/ufw/before.rules
```

**Add NAT rules at the top of the file (before the `*filter` section):**
```
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]

# Forward traffic through wg0 and masquerade source IPs
-A POSTROUTING -s 192.168.1.0/24 -o wg0 -j MASQUERADE

# Don't delete the 'COMMIT' line or these rules won't be processed
COMMIT
```

**Important:** Replace `192.168.1.0/24` with your actual subnet.

**Restart UFW to apply changes:**
```bash
sudo ufw disable && sudo ufw enable
```

**Verify WireGuard status:**
```bash
sudo wg show
```

Should show active connection with recent handshake.

**Example output:**
```
interface: wg0
  public key: [YOUR_PUBLIC_KEY]
  private key: (hidden)
  listening port: 43424
  fwmark: 0xca6c

peer: [VPN_SERVER_PUBLIC_KEY]
  endpoint: [VPN_SERVER_IP]:51820
  allowed ips: 0.0.0.0/0
  latest handshake: 25 seconds ago
  transfer: 1.67 MiB received, 639.98 KiB sent
```

**Verify Pi routes through VPN:**
```bash
curl ifconfig.me
```

Should display ProtonVPN server IP (e.g., `185.x.x.x`).

---

## Step 5: Client Configuration (Laptop/Desktop)

### Disable IPv6 on Client

IPv6 can bypass the VPN gateway, causing IP leaks.

**Check current IP (should show IPv6 if enabled):**
```bash
curl ifconfig.me
```

**Temporarily disable IPv6:**
```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.<INTERFACE>.disable_ipv6=1  # Replace <INTERFACE> with your interface name (e.g., wlp3s0)
```

**Verify IPv6 is disabled:**
```bash
ip a show <INTERFACE>  # No inet6 addresses should appear
curl ifconfig.me       # Should now show IPv4 address
```

**Make IPv6 disable permanent (optional):**
```bash
sudo vim /etc/sysctl.d/40-ipv6.conf
```

Add:
```ini
# Disable IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Apply:
```bash
sudo sysctl -p /etc/sysctl.d/40-ipv6.conf
```

### Configure Permanent VPN Gateway Routing

**Configure NetworkManager to route through the Pi:**
```bash
nmcli connection modify "YOUR_WIFI_NAME" ipv4.method auto
nmcli connection modify "YOUR_WIFI_NAME" ipv4.ignore-auto-routes yes
nmcli connection modify "YOUR_WIFI_NAME" +ipv4.routes "0.0.0.0/0 192.168.1.100"
nmcli connection up "YOUR_WIFI_NAME"
```

**Note:** Replace:
- `YOUR_WIFI_NAME` with your actual WiFi connection name
- `192.168.1.100` with your Pi's static IP address

**What these commands do:**
- `ipv4.method auto` - Use DHCP for IP address, subnet mask, and DNS
- `ipv4.ignore-auto-routes yes` - Ignore the default gateway provided by DHCP
- `+ipv4.routes "0.0.0.0/0 192.168.1.100"` - Set Pi as the default gateway for all traffic
- `nmcli connection up` - Apply changes immediately

### Verify Gateway Configuration

**Check routing table:**
```bash
ip route show default
```

Should show:
```
default via 192.168.1.100 dev wlp3s0 proto static metric 600
```

**Verify traffic routes through VPN:**
```bash
curl ifconfig.me
```

Should display ProtonVPN server IP (not your ISP IP).

**Test DNS resolution:**
```bash
ping google.com
```

**Test web browsing:**
Open a browser and verify internet connectivity.

---

## How It Works

### Traffic Flow
```
[Client Device]
    ↓ (192.168.1.x)
[Pi wlan0 - receives traffic]
    ↓ (IP forwarding enabled)
[Pi FORWARD chain - allows traffic through]
    ↓
[Pi POSTROUTING - MASQUERADE rewrites source IP]
    ↓ (10.2.0.2 - VPN IP)
[Pi wg0 - encrypted VPN tunnel]
    ↓
[ProtonVPN Server]
    ↓
[Internet]
```

### Key Concepts Explained

**VPN Gateway:**
A device that routes network traffic through an encrypted VPN tunnel, providing privacy and security for all connected devices.

**IP Forwarding:**
Allows the Pi to pass network packets between interfaces (wlan0 → wg0), acting as a router rather than just an endpoint.

**NAT (Network Address Translation):**
Rewrites packet source IP addresses so multiple devices can share a single public IP. Implemented via MASQUERADE in iptables.

**MASQUERADE:**
A specific type of NAT that dynamically rewrites source IPs to match the outgoing interface's IP (ideal for VPNs with dynamic IPs).

**Netfilter:**
The Linux kernel's packet filtering framework. Network packets pass through 5 "hooks":
1. `PREROUTING` - Packet just arrived
2. `INPUT` - Packet destined for this machine
3. `FORWARD` - Packet passing through (gateway function)
4. `OUTPUT` - Packet leaving this machine
5. `POSTROUTING` - Packet about to exit the network

**iptables:**
The userspace tool for configuring netfilter rules at each hook point.

**iptables Tables:**
- `filter` - Decides whether to allow or block packets
- `nat` - Changes source/destination IP addresses and ports
- `mangle` - Modifies packet headers (TTL, QoS, etc.)

**Default Gateway:**
The IP address where packets are sent when the destination is not on the local network. Normally your router; in this setup, it's the Pi.

**0.0.0.0/0 Route:**
A catch-all route meaning "all destinations." Packets matching this route (everything not specifically routed elsewhere) go to the specified gateway.

---

## Troubleshooting

### WireGuard won't start
**Check service status:**
```bash
sudo systemctl status wg-quick@wg0
sudo journalctl -u wg-quick@wg0 -n 50
```

**Common issues:**
- IPv6 configuration in `AllowedIPs` when IPv6 is disabled
- Missing `resolvconf` package
- Malformed configuration file

### Traffic not routing through VPN
**Verify FORWARD policy:**
```bash
sudo iptables -L FORWARD -v
```

**Check NAT rules:**
```bash
sudo iptables -t nat -L POSTROUTING -v -n
```

Should show MASQUERADE rule for subnet → wg0.

**Verify IP forwarding:**
```bash
sysctl net.ipv4.ip_forward  # Should be 1
```

### Client can't reach internet after gateway change
**Check if Pi's VPN is up:**
```bash
ssh pi@192.168.1.100
sudo wg show
curl ifconfig.me
```

**Verify client's routing:**
```bash
ip route show default
ping 192.168.1.100  # Can you reach the Pi?
ping 8.8.8.8        # Can you reach internet IPs?
nslookup google.com # Does DNS work?
```

### IPv6 leaks
**Disable IPv6 on all interfaces:**
```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
ip a | grep inet6  # Should show no addresses
```

**Force curl to use IPv4:**
```bash
curl -4 ifconfig.me
```

---

## Verification Checklist

- [ ] WireGuard service is active and enabled
- [ ] `sudo wg show` displays active connection
- [ ] Pi's public IP shows ProtonVPN server (not ISP)
- [ ] IP forwarding enabled (`sysctl net.ipv4.ip_forward` = 1)
- [ ] UFW FORWARD policy set to ACCEPT
- [ ] MASQUERADE rule present in iptables NAT table
- [ ] Client routing table shows Pi as default gateway
- [ ] Client's public IP shows ProtonVPN server
- [ ] Client can browse internet normally
- [ ] DNS resolution works on client

---

## Skills Practiced

- VPN configuration (WireGuard/ProtonVPN)
- Network routing and gateway configuration
- NAT/MASQUERADE implementation
- iptables/UFW firewall management
- IPv4/IPv6 networking concepts
- systemd service management
- NetworkManager CLI (nmcli) usage
- Log analysis and troubleshooting
- Kernel parameter configuration (sysctl)

---

## References

- [ProtonVPN WireGuard Guide](https://protonvpn.com/blog/openvpn-vs-wireguard)
- [Raspberry Pi VPN Gateway Tutorial](https://raspberrytips.com/raspberry-pi-vpn-gateway/)
- [Debian WireGuard Wiki](https://wiki.debian.org/WireGuard)
- [Red Hat Firewall Forwarding Guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/4/html/security_guide/s1-firewall-ipt-fwd)
- [iptables NAT/MASQUERADE Concepts](https://www.geeksforgeeks.org/linux-unix/using-masquerading-with-iptables-for-network-address-translation-nat/)
- [Linux IP Forwarding Guide](https://pimylifeup.com/linux-ip-forwarding/)
- [UFW Port Forwarding](https://tecadmin.net/setup-port-forwarding-using-ufw/)
- [IP Command Static Routes](https://www.linuxtechi.com/add-delete-static-route-linux-ip-command/)
