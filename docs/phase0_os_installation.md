## Phase 0: OS Installation

### Goal

Install Raspberry Pi OS Lite to an external SSD and gain SSH access.

### Steps Performed

1. **Verified drives**

```bash
lsblk
```

2. **Made Raspberry Pi Imager executable**

```bash
sudo chmod a+x imager_2.0.0_amd64.AppImage
```

3. **Ran Raspberry Pi Imager**

```bash
./imager_2.0.0_amd64.AppImage
```

4. **Configured OS options**

* Set hostname
* Created user/password
* Enabled SSH
* Configured Wi-Fi

5. **Scanned network to find the Pi**

```bash
nmap -sn 192.168.x.x/24
```

6. **SSH into the Pi**

```bash
ssh joe@192.168.x.x
```

7. **Cleaned old SSH host keys (if needed)**

```bash
ssh-keygen -R 192.168.x.x
```

### Mistakes & Recovery

* Initially used `dd` incorrectly by writing the **compressed `.xz` image** directly
* Tried manual headless Wi-Fi setup (`wpa_supplicant.conf`) but couldn’t reliably discover the Pi
* Switched to Raspberry Pi Imager for predictable setup

### What I Learned

* Difference between compressed vs raw disk images
* Why Raspberry Pi Imager simplifies headless installs
* How to safely identify and write to block devices

### References

* [https://www.raspberrypi.com/documentation/](https://www.raspberrypi.com/documentation/)
* [https://pendrivelinux.com/create-bootable-usb-from-iso-using-dd/](https://pendrivelinux.com/create-bootable-usb-from-iso-using-dd/)
* [https://learn.sparkfun.com/tutorials/headless-raspberry-pi-setup/wifi-with-dhcp](https://learn.sparkfun.com/tutorials/headless-raspberry-pi-setup/wifi-with-dhcp)
