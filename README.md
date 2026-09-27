# wowlan-orangepi-nitro
A lightweight bash script to remotely wake up an Acer Nitro laptop over Wi-Fi (WoWLAN) from an Orange Pi via Tailscale and monitor status with mDNS

# Remote Wireless Wake-on-LAN (WoWLAN) | Orange Pi PC ➔ Acer Nitro

A minimal infrastructure solution to remotely wake up an Acer Nitro laptop over Wi-Fi (WoWLAN) using an Orange Pi PC as a 24/7 gateway via Tailscale, resolving local dynamic IPs using mDNS.

> ⚠️ **Why just the script isn't enough:** Waking a modern laptop over Wi-Fi under Linux requires aligning BIOS power management, wireless kernel states, specific Realtek driver fixes on Ubuntu 24.04, and mDNS resolver orders. This document outlines the complete research and system configuration.

---

## 🏗️ Architectural Overview

[ External Device (Phone/Laptop) ] (Via Tailscale VPN - Global Access)
▼
[ Orange Pi PC (24/7) ] ➔ (Sends UDP Magic Packet over local Wi-Fi)
▼
[ Acer Nitro Laptop (Wakes up instantly) ]

---

## 🛠️ Step-by-Step System Configuration

### 1. Target Machine: Acer Nitro Firmware (BIOS)
By default, the Acer Nitro cuts power to network interfaces when sleeping to save battery. 
1. Reboot your laptop and press **F2** repeatedly to enter the BIOS setup.
2. Navigate to the **Main** or **Advanced** tab using the arrow keys.
3. Locate **Wake on LAN** and change it to `Enabled`.
4. Disable any option related to **Power Saving LAN** or **Deep Sleep (ERP / ErP Ready)**.
5. Press **F10** to save, select *Yes*, and hit Enter.

### 2. Target Machine: OS Level (Ubuntu 24.04 LTS)

#### A. The Realtek `r8169` Driver Fix
The default Linux kernel driver (`r8169`) has a known bug with the Realtek RTL8111/8168 chip family on Ubuntu 24.04, causing interfaces to get stuck in an `unavailable` state.
```bash
# Install the official proprietary DKMS module
sudo apt update && sudo apt install -y r8168-dkms

# Permanently blacklist the broken default driver
echo "blacklist r8169" | sudo tee /etc/modprobe.d/blacklist-r8169.conf
```

#### B. Persist Wireless Wake-on-LAN (WoWLAN)
Wireless cards drop connections during sleep unless explicitly forced into listening mode. Create a systemd service to persist this setting on boot:
```bash
sudo nano /etc/systemd/system/persistir-wowlan.service
```
Add the following content:
```ini
[Unit]
Description=Persist Wake-on-Wireless-LAN (WoWLAN)
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iw phy0 wowlan enable magic-packet
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Enable and start the service:
```bash
sudo systemctl daemon-reload && sudo systemctl enable --now persistir-wowlan.service
```

#### C. Firewall Rule
Allow mDNS requests (UDP port 5353) through the local firewall so the gateway can resolve the hostname:
```bash
sudo ufw allow 5353/udp comment 'Allow Avahi mDNS'
```

---

### 3. Gateway Machine: Orange Pi PC Configuration

#### A. Lock Avahi to Wi-Fi
To prevent the Avahi daemon from freezing when physical network cables are unplugged, force it to listen exclusively to your wireless interface (`wlan0`):
```bash
sudo nano /etc/avahi/avahi-daemon.conf
```
Under the `[server]` section, modify or add the following lines:
```ini
allow-interfaces=wlan0
deny-interfaces=eth0
```
Restart the service to apply changes: 
```bash
sudo systemctl restart avahi-daemon
```

#### B. Fix the Name Service Switch (mDNS Resolution)
If your Orange Pi cannot ping `.local` addresses, the OS is missing the mDNS resolution module in its network lookup order.
```bash
sudo apt install -y libnss-mdns
sudo nano /etc/nsswitch.conf
```
Find the `hosts:` line and insert `mdns4_minimal [NOTFOUND=return]` right before `dns`:
```text
hosts:          files mymachines mdns4_minimal [NOTFOUND=return] dns myhostname
```

---

## 📜 The Automation Script (`acordar_nitro.sh`)
Place this script on your Orange Pi PC. It triggers the magic packet, enters a loop pinging the target via mDNS for up to 60 seconds, and confirms when the machine is ready for SSH or Claude CLI access.

```bash
#!/bin/bash

# ========================================================
# CONFIGURATION
# ========================================================
MAC_NITRO="50:2e:91:d7:6a:82"
HOST_NITRO="linwell1.local"
TIMEOUT=60
INTERVAL=2

echo "====== 🧠 Wireless Wake-on-LAN Process Started ======"
echo "📢 Sending Magic Packet to Nitro (\$MAC_NITRO)..."

# Send the broadcast magic packet over the Wi-Fi network
wakeonlan \$MAC_NITRO

echo "⏳ Waiting for Nitro to boot up and reply..."
echo "🔍 Monitoring '\$HOST_NITRO' for up to \$TIMEOUT seconds..."

elapsed=0

while [ \$elapsed -lt \$TIMEOUT ]; do
    ping -c 1 -W 1 \$HOST_NITRO > /dev/null 2>&1
    if [ \$? -eq 0 ]; then
        echo ""
        echo "========================================================"
        echo "🚀 SUCCESS! Nitro (\$HOST_NITRO) is ONLINE and ready!"
        echo "   Total boot time: \$elapsed seconds."
        echo "========================================================"
        exit 0
    fi
    printf "."
    sleep \$INTERVAL
    elapsed=\$((elapsed + INTERVAL))
done

echo ""
echo "========================================================"
echo "❌ FAILURE: Target did not wake up within 1 minute."
echo "========================================================"
exit 1
```

## ⚙️ How to use
Give execution permissions to the script on your Orange Pi:
```bash
chmod +x acordar_nitro.sh
```
Run it anytime from your phone (via Terminus) or another computer connected to your Tailscale network:
```bash
./acordar_nitro.sh
```
