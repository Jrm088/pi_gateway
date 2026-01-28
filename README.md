# Raspberry Pi Security Gateway

## Project Motivation

After completing the **RHCSA Udemy course by Imran Afzal**, I wanted hands-on experience beyond lab exercises. This project documents a fully functional security gateway I built and use personally, highlighting mistakes, recovery steps, and lessons learned along the way.

The gateway provides:

* Network-wide ad blocking via **Pi-hole**
* VPN routing for all devices using **ProtonVPN**
* Intrusion prevention with **Fail2ban + CrowdSec**
* Real-time monitoring via **Netdata + Uptime Kuma**
* Firewall management using **UFW / iptables**

Each service demonstrates practical sysadmin work: service management, log analysis, networking, security hardening, and troubleshooting.

## Hardware

* Raspberry Pi 4 (8GB)
* 240GB SSD
* SATA → USB 3.0 adapter
* Arch Linux laptop (client machine)

## Documentation

The project is organized into phases, each fully documented in the `docs/` folder:

- [Phase 0 – OS Installation](docs/phase0_os_installation.md)
- [Phase 1 – System Hardening](docs/phase1_system_hardening.md)
- [Phase 2 – Pi-hole](docs/phase2_pihole.md)
- [Phase 3 – VPN Gateway](docs/phase3_vpn_gateway.md)
- [Phase 4 – Security Services](docs/phase4_security_services.md)
- [Phase 5 – Monitoring](docs/phase5_monitoring.md)
