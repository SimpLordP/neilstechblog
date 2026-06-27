---
title: "Automating Raspberry Pi Node Provisioning with Ansible"
date: 2026-06-20
draft: false
tags: ["DevOps", "Ansible", "Automation", "Raspberry Pi", "Homelab"]
series: ["Homelab Hero"]
description: "Using Ansible to automate the repetitive setup tasks across a Raspberry Pi K3s cluster from package installation to disabling WiFi - instead of SSHing into every node manually."
---

After provisioning my fourth Raspberry Pi node by hand, I had the same realization every infrastructure engineer eventually has: doing the same setup steps manually four times is a bug, not a feature.

This post covers using Ansible to automate the baseline configuration tasks across all nodes in a Raspberry Pi K3s cluster - package installation, WiFi disabling, cgroup configuration, and Docker setup - from a single playbook run.

# Pre-requisites

- Ansible installed on your control machine (WSL in my case)
- SSH key-based authentication already configured on each Pi node
- An inventory of node IP addresses or hostnames

# Installing Ansible

On your WSL control node:

```bash
sudo apt update
sudo apt install -y ansible
```

Verify the install:

```bash
ansible --version
```

# Building the Inventory
Create a dedicated directory for your Ansible project:

```bash
mkdir -p ~/homelab-platform/ansible
cd ~/homelab-platform/ansible
```

Create an inventory file listing your nodes:

```bash
nano inventory.ini
```

Paste this, adjusting IPs to match your network:

```ini
[control_plane]
pi4-control-plane ansible_host=10.252.1.201

[workers]
pi4-worker-1 ansible_host=10.252.1.202
pi4-worker-2 ansible_host=10.252.1.203
pi4-worker-3 ansible_host=10.252.1.204

[all:vars]
ansible_user=npodoba
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Test connectivity to all nodes:

```bash
ansible all -i inventory.ini -m ping
```

You should see a `pong` response from each node.

# Writing the Provisioning Playbook

Create the playbook file:

```bash
nano provision-pi.yml
```

Paste this:

```yaml
---
- name: Provision Raspberry Pi K3s nodes
  hosts: all
  become: true
  tasks:

    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Upgrade all packages
      apt:
        upgrade: dist

    - name: Install core packages
      apt:
        name:
          - curl
          - wget
          - git
          - htop
          - net-tools
          - nmap
          - unzip
          - rfkill
        state: present

    - name: Disable WiFi via rfkill
      command: rfkill block wifi
      changed_when: false

    - name: Bring down wlan0 interface
      command: ip link set wlan0 down
      changed_when: false
      ignore_errors: true

    - name: Deny DHCP usage for wlan0
      lineinfile:
        path: /etc/dhcpcd.conf
        line: "denyinterfaces wlan0"
        create: yes

    - name: Stop wpa_supplicant service
      systemd:
        name: wpa_supplicant
        state: stopped
        enabled: no
      ignore_errors: true

    - name: Configure cgroup parameters for Kubernetes
      lineinfile:
        path: /boot/firmware/cmdline.txt
        backrefs: yes
        regexp: '^(.*)$'
        line: '\1 cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1'

    - name: Install Docker
      shell: curl -fsSL https://get.docker.com | sh
      args:
        creates: /usr/bin/docker

    - name: Add user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes

    - name: Reboot to apply cgroup changes
      reboot:
        msg: "Rebooting to apply Ansible-managed cgroup configuration"
        reboot_timeout: 300
```

# Running the Playbook

Execute against all nodes:

```bash
ansible-playbook -i inventory.ini provision-pi.yml
```

To target just the worker nodes:

```bash
ansible-playbook -i inventory.ini provision-pi.yml --limit workers
```

To do a dry run without making changes:

```bash
ansible-playbook -i inventory.ini provision-pi.yml --check
```

# Verifying the Results

After the playbook completes and nodes reboot, verify the cgroup configuration applied correctly:

```bash
ansible all -i inventory.ini -m shell -a "cat /sys/fs/cgroup/cgroup.controllers"
```

Confirm WiFi is disabled across the fleet:

```bash
ansible all -i inventory.ini -m shell -a "rfkill list"
```

Confirm Docker installed successfully:

```bash
ansible all -i inventory.ini -m shell -a "docker --version"
```

# Why This Matters

Running this playbook once took under two minutes to provision all four nodes identically. Doing the same steps manually across four Pi units the first time took the better part of an afternoon, with at least one node ending up slightly different from the others due to a missed step.

This is the actual value proposition of configuration management, not the time savings on the first run, but the guarantee that node five, ten, or fifty will be configured exactly the same way as node one.

# FAQ

## Why use Ansible instead of a bash script?

A bash script run loop would technically work for this scale, but Ansible gives you idempotency for free. Running the playbook a second time against an already-configured node makes no changes and reports nothing as failed, where a bash script would re-run every command regardless of current state.

## Do I need an Ansible control node running 24/7?

No. Ansible is agentless and only needs to run from your workstation or laptop when you actually want to push configuration changes. The control node does not need to stay online afterward.

## How do I add a fifth node later?

Add a new host entry to inventory.ini under the appropriate group, then re-run the playbook with --limit targeting just the new host.
