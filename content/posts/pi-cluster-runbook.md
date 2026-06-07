---
title: "Raspberry Pi Cluster Runbook - Phase 1: Node Provisioning"
date: 2026-06-05
draft: false
tags: ["Homelab", "Raspberry Pi", "K3s", "Kubernetes", "Docker", "Networking"]
series: ["Homelab Build"]
description: "A reproducible baseline runbook for provisioning Raspberry Pi 4 nodes to support a K3s Kubernetes cluster."
---

## Objective

Establish a consistent, reproducible baseline for Raspberry Pi nodes to support a future Kubernetes (k3s) cluster. This phase focuses on hardware setup, OS provisioning, networking, and container runtime readiness.

---

## Architecture Overview

* Nodes:
  * 1x Control Plane (Pi4)
  * 3x Worker Nodes (Pi4)
* Network:
  * Linksys Router (DHCP)
  * Netgear GS308PP Switch
* Access:
  * SSH via Ubuntu (WSL) on Windows host
* OS:
  * Debian-based Raspberry Pi OS

---

## Phase 1: Hardware Setup

### Components

* Raspberry Pi 4 devices
* MicroSD cards (32GB–256GB)
* Netgear GS308PP switch
* Ethernet cables (Cat5e/Cat6)
* USB-C power supply (multi-port Anker recommended)

### Physical Topology

```text
Router (Linksys)
    ↓
Switch (Netgear GS308PP)
    ↓
Pi Nodes (eth0)
    ↓
Windows Desktop (WSL access)
```

### Key Notes

* Switch is unmanaged → no configuration required
* All Pis connected via Ethernet (eth0)
* WiFi intentionally disabled for cluster stability

---

## Phase 2: OS Installation

### Tool Used

* Raspberry Pi Imager (Windows)

### OS Selection

* Raspberry Pi OS (64-bit, Lite preferred)

### Advanced Configuration (IMPORTANT)

* Enable SSH
* Set username/password
* Configure hostname per node:
  * `pi4-control-plane`
  * `pi4-worker-1`
  * `pi4-worker-2`
  * `pi4-worker-3`

### Validation

After flashing:

* Boot Pi
* Confirm activity LED (green blinking)
* Verify network visibility via `nmap`

---

## Phase 3: Network Discovery

From Ubuntu (WSL):

```bash
nmap -sn 10.252.1.0/24
```

Identify nodes via:

* Hostnames
* IP addresses assigned via DHCP

---

## Phase 4: SSH Access

```bash
ssh @
```

If host key mismatch occurs:

```bash
ssh-keygen -f '~/.ssh/known_hosts' -R ''
```

---

## Phase 5: Baseline System Configuration

### Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Core Tools

```bash
sudo apt install -y curl wget git htop net-tools nmap unzip rfkill
```

---

## Phase 6: Kernel Configuration (cgroups)

Edit:

```bash
sudo nano /boot/firmware/cmdline.txt
```

Ensure the line contains:

```text
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

Reboot:

```bash
sudo reboot
```

Verify:

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

Expected:

```text
cpuset cpu io memory pids
```

---

## Phase 7: Disable WiFi (Critical for Cluster Stability)

### Immediate Disable

```bash
sudo rfkill block wifi
sudo ip link set wlan0 down
```

### Prevent DHCP Usage

```bash
echo 'denyinterfaces wlan0' | sudo tee -a /etc/dhcpcd.conf
```

### Disable WiFi Services

```bash
sudo systemctl stop wpa_supplicant
sudo systemctl disable wpa_supplicant
```

### Persist Disable via systemd

```bash
sudo tee /etc/systemd/system/disable-wifi.service > /dev/null <<EOF
[Unit]
Description=Disable WiFi
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/rfkill block wifi

[Install]
WantedBy=multi-user.target
EOF
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable disable-wifi.service
```

### Verification

```bash
ip a
rfkill list
```

Expected:

* `eth0` active with IP
* `wlan0` no IP
* `Soft blocked: yes`

---

## Phase 8: Docker Installation

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Add user to Docker group:

```bash
sudo usermod -aG docker $USER
```

Reconnect session and verify:

```bash
docker run hello-world
```

---

## Phase 9: Network Validation

Check interfaces:

```bash
ip a
ip route
```

Confirm:

* Single interface (`eth0`)
* No WiFi routing
* Default gateway via router

---

## Phase 10: Troubleshooting Lessons Learned

### Issue: SSH "Host Identification Changed"

* Cause: Reimaged device or IP reassignment
* Fix:

```bash
ssh-keygen -R 
```

---

### Issue: Multiple IPs per Node

* Cause: WiFi + Ethernet both active
* Impact:
  * Routing inconsistencies
  * Cluster instability
* Fix: Disable WiFi completely

---

### Issue: Docker Permission Denied

* Cause: User not in docker group
* Fix:

```bash
sudo usermod -aG docker $USER
```

---

### Issue: dpkg Lock Errors

* Cause: Background apt process
* Fix: Wait or complete pending configuration

```bash
sudo dpkg --configure -a
```

---

## Phase 1 Completion Criteria

All nodes must meet:

* [x] Reachable via SSH
* [x] Unique hostname assigned
* [x] System updated
* [x] WiFi disabled permanently
* [x] Single network interface (eth0)
* [x] Time synchronized
* [x] cgroups enabled (memory present)
* [x] Docker installed and functional

---

## Next Phase

Phase 2 will include:

* Static IP assignment
* SSH key-based authentication
* k3s control plane deployment
* Worker node cluster join
