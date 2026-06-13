---
title: "Raspberry Pi 4 Provisioning SOP"
date: 2026-06-04
draft: false
tags: ["Homelab", "Raspberry Pi", "Linux", "Docker", "Infrastructure"]
series: ["Homelab Build"]
description: "A production-grade standard operating procedure for provisioning Raspberry Pi 4 nodes for a K3s cluster."
---

# Raspberry Pi 4 Provisioning SOP (Windows → Infrastructure)

## Objective

Provision a Raspberry Pi 4 as a remotely accessible Linux node, install Docker, and deploy a containerized service using Docker Compose.

---

# 1. Hardware + Network Setup

## Components

* Raspberry Pi 4
* MicroSD card (≥32GB, 256GB used)
* Ethernet cable
* Netgear GS308PP switch
* Windows desktop (host machine)

## Physical Connections

1. Connect Pi → Netgear switch (Ethernet)
2. Connect switch → router (Linksys)
3. (Optional) Connect desktop → switch

## Expected Outcome

* Pi receives IP via DHCP from router
* Network LEDs active (green = link, blinking = activity)

---

# 2. OS Installation (Windows)

## Tooling

* Raspberry Pi Imager (Windows)

## Steps

1. Insert MicroSD into Windows machine
2. Open Raspberry Pi Imager
3. Select:
   * OS: Debian-based image (Debian 13 used)
   * Storage: MicroSD card
4. Configure (advanced options):
   * Enable SSH
   * Set username/password
   * Configure hostname (e.g., `pi4-control-plane`)
5. Write image to disk

## Validation

* `bootfs` partition appears after write
* No write errors

---

# 3. First Boot + Network Discovery

## Power On

* Insert SD card into Pi
* Connect power

## On Windows (PowerShell)

```powershell
ipconfig
arp -a
```

## On Router (preferred)

* Log into router UI (Linksys)
* Identify DHCP lease for Pi

## Expected Outcome

* Pi assigned IP (e.g., `10.252.1.201`)

---

# 4. SSH Access

## From Windows or WSL

```bash
ssh <user>@<pi-ip>
```

## First Connection

* Accept host fingerprint
* Enter password

## Validation

```bash
hostname
whoami
hostname -I
```

---

# 5. System Validation

```bash
cat /etc/os-release
uname -a
```

## Expected

* Debian 13 (trixie)
* Hostname matches configured value

---

# 6. Package Management Stabilization

## Issue Encountered

* `dpkg` lock due to background updates

## Resolution Process

```bash
ps -p <pid> -f
top
pgrep -a apt
pgrep -a dpkg
```

### Correct Handling

* Wait for processes to complete
* Avoid removing lock files

### Recovery (if stuck)

```bash
sudo kill -9 <pid>
sudo dpkg --configure -a
sudo apt update
```

---

# 7. Docker Installation

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

## Configure Non-Root Access

```bash
sudo usermod -aG docker $USER
exit
ssh <user>@<pi-ip>
```

## Validation

```bash
docker run hello-world
```

---

# 8. Project Initialization

```bash
mkdir -p ~/homelab-platform
cd ~/homelab-platform
```

## Install Git

```bash
sudo apt install -y git
git config --global user.name "<name>"
git config --global user.email "<email>"
```

---

# 9. GitHub SSH Setup

```bash
ssh-keygen -t ed25519 -C "<email>"
cat ~/.ssh/id_ed25519.pub
```

## Add key to GitHub → Settings → SSH and GPG Keys

## Validate

```bash
ssh -T git@github.com
```

---

# 10. First Service Deployment (Nginx)

## Test Container

```bash
docker run -d --name nginx-test -p 8080:80 nginx
```

## Validate

* Browser: `http://<pi-ip>:8080`

---

# 11. Transition to Docker Compose

## docker-compose.yml

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx-test
    ports:
      - "8080:80"
    restart: unless-stopped
```

## Resolve Conflict (if needed)

```bash
docker rm -f nginx-test
```

## Deploy

```bash
docker compose up -d
docker ps
```

---

# 12. Repository Integration

```bash
git init
git remote add origin git@github.com:<username>/homelab-platform.git
git add .
git commit -m "Initial homelab setup with nginx service"
git push -u origin main
```

---

# Troubleshooting Reference

## Blink Pi to Locate Faulty Nodes

```bash
echo none | sudo tee /sys/class/leds/led0/trigger
while true; do echo 1 | sudo tee /sys/class/leds/led0/brightness; sleep 0.5; echo 0 | sudo tee /sys/class/leds/led0/brightness; sleep 0.5; done
```

### Once discovered

```bash
Ctrl + C
echo mmc0 | sudo tee /sys/class/leds/led0/trigger
```

---

## No IP Address (169.254.x.x)

* Cause: No DHCP
* Fix: Ensure switch → router connection

---

## Cannot SSH (Connection Refused)

```bash
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

## Docker Permission Denied

```bash
sudo usermod -aG docker $USER
```

---

## Error Cannot Write to Disk

## Port Not Accessible

```bash
docker ps
docker logs <container>
```

---

# Final State

* Pi accessible via SSH
* Docker installed and functional
* GitHub integration complete
* Service deployed via Docker Compose
* Node designated as `pi4-control-plane`
