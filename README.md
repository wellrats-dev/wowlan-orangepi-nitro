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
By default, laptop mainboards completely cut power to wireless chips when entering sleep mode. We must explicitly force the hardware to keep the Wi-Fi card in listening mode:
1. Reboot the laptop and press **F2** repeatedly to enter the BIOS setup.
2. Navigate to the **Main** or **Advanced** tab using the arrow keys.
3. Locate **Wake on LAN** and change it to `Enabled`.
4. Press **F10** to save, select *Yes*, and hit Enter.

### 2. Target Machine: OS Level (Ubuntu 24.04 LTS)

#### A. Persist Wireless Wake-on-LAN (WoWLAN) Gating
Wireless cards drop network states during sleep. We need a systemd service to inject the `magic-packet` listener capability onto the wireless card at boot time:
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
Enable and apply the service instantly:
```bash
sudo systemctl daemon-reload && sudo systemctl enable --now persistir-wowlan.service
```

#### B. Firewall Configuration (optional)
If you wants llow local mDNS requests (UDP port 5353) through the local firewall so the gateway can ping and find the laptop without knowing its dynamic IP address:
```bash
sudo ufw allow 5353/udp comment 'Allow Avahi mDNS'
```

---

### 3. Gateway Machine: Orange Pi PC Configuration (Ubuntu Jammy)

#### A. Bind Avahi Daemon to the Wireless Interface  (optional)
Force the Avahi resolution service to strictly listen to your wireless interface (`wlan0`) to ensure fast local broadcast resolution:
```bash
sudo nano /etc/avahi/avahi-daemon.conf
```
Under the `[server]` section, modify or add the following lines:
```ini
allow-interfaces=wlan0
```
Restart the service to apply changes: 
```bash
sudo systemctl restart avahi-daemon
```

#### B. Fix the Name Service Switch (mDNS Resolution)  (optional)
If your Orange Pi cannot resolve `.local` domains, the operating system is missing the mDNS resolution module in its local hostname lookup priorities.
```bash
sudo apt update && sudo apt install -y libnss-mdns
sudo nano /etc/nsswitch.conf
```
Find the line starting with `hosts:` and insert `mdns4_minimal [NOTFOUND=return]` right before `dns`:
```text
hosts:          files mymachines mdns4_minimal [NOTFOUND=return] dns myhostname
```

---

## 📜 The Automation Script (`wake_nitro.sh`)
Place this script on your Orange Pi PC. It triggers the magic packet into the air, enters a loop pinging the target machine via mDNS for up to 60 seconds, and visually confirms when the machine is awake and ready for SSH or Claude CLI access.

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

## ⚙️ How to execute
Give execution permissions to the script on your Orange Pi:
```bash
chmod +x wake_nitro.sh
```
Run it anytime from your phone (via Terminus) or another computer connected to your Tailscale network:
```bash
./wake_nitro.sh
```
