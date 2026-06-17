---
title: "Backing Up Your Homelab to AWS S3 with the AWS CLI"
date: 2026-06-16
draft: false
tags: ["AWS", "S3", "Backup", "CLI", "Homelab"]
series: ["AWS from the Homelab"]
description: "Using the AWS CLI to back up critical homelab data from a Raspberry Pi cluster to S3. A practical guide to cloud backup for edge devices."
---

Your Raspberry Pi runs 24/7 and stores data you care about. MicroSD cards fail. What happens to your data when one does?

In this post we'll use the AWS CLI to back up homelab data from a Pi node directly to S3. No third party tools, no agents - just the AWS CLI and a bash script running on a systemd timer.

# Pre-requisites

- AWS account with S3 access
- AWS CLI installed on your Pi node
- IAM user with S3 write permissions
- Data on your Pi you actually want to back up

# Setting Up the AWS CLI on Raspberry Pi

Install the AWS CLI on your Pi node:

```bash
sudo apt update
sudo apt install -y awscli
```

Verify the install:

```bash
aws --version
```

Configure your credentials:

```bash
aws configure
```

You will be prompted for:

AWS Access Key ID: <your-key>

AWS Secret Access Key: <your-secret>

Default region name: us-east-1

Default output format: json

# Creating Your S3 Bucket

Create a dedicated bucket for homelab backups:

```bash
aws s3 mb s3://your-homelab-backups --region us-east-1
```

Enable versioning so you can recover previous versions of files:

```bash
aws s3api put-bucket-versioning \
    --bucket your-homelab-backups \
    --versioning-configuration Status=Enabled
```

# Copy vs Sync

The AWS CLI gives you two main commands for moving data to S3.

| Use Case | Command | Why |
| --- | --- | --- |
| Initial backup | `cp` | Safe, won't delete anything |
| Regular backups | `sync` | Mirrors source to destination |
| Archive old files | `cp` | Preserves historical data |
| One-time transfer | `cp` | Simple and safe |

## aws s3 cp

Copies a single file or directory to S3:

```bash
# Copy a single file
aws s3 cp /home/npodoba/important-file.txt s3://your-homelab-backups/

# Copy a directory recursively
aws s3 cp /home/npodoba/homelab-platform/ s3://your-homelab-backups/homelab-platform/ --recursive
```

## aws s3 sync

Syncs a local directory to S3, only transferring new or changed files:

```bash
aws s3 sync /home/npodoba/homelab-platform/ s3://your-homelab-backups/homelab-platform/
```

Always test with --dryrun first:

```bash
aws s3 sync /home/npodoba/homelab-platform/ s3://your-homelab-backups/homelab-platform/ --dryrun
```

# Building the Backup Script

Create a reusable backup script:

```bash
nano /home/npodoba/scripts/s3-backup.sh
```

Paste this into the script:

```bash
#!/bin/bash

BUCKET="your-homelab-backups"
SOURCE="/home/npodoba/homelab-platform"
LOG="/var/log/s3-backup.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$DATE] Starting backup..." >> $LOG

aws s3 sync $SOURCE s3://$BUCKET/homelab-platform/ \
    --exclude "*.tmp" \
    --exclude ".git/*" \
    --delete \
    >> $LOG 2>&1

if [ $? -eq 0 ]; then
    echo "[$DATE] Backup completed successfully." >> $LOG
else
    echo "[$DATE] Backup failed." >> $LOG
fi
```

Make it executable:

```bash
chmod +x /home/npodoba/scripts/s3-backup.sh
```

# Automating with systemd

Create a systemd service unit:

```bash
sudo nano /etc/systemd/system/s3-backup.service
```

Paste into the service file:

```ini
[Unit]
Description=S3 Homelab Backup
After=network.target

[Service]
Type=oneshot
User=npodoba
ExecStart=/home/npodoba/scripts/s3-backup.sh

[Install]
WantedBy=multi-user.target
```

Create the timer:

```bash
sudo nano /etc/systemd/system/s3-backup.timer
```

Paste into the timer file:

```ini
[Unit]
Description=Run S3 backup every 6 hours

[Timer]
OnBootSec=10min
OnUnitActiveSec=6h

[Install]
WantedBy=timers.target
```

Enable and start the timer:

```bash
sudo systemctl daemon-reload
sudo systemctl enable s3-backup.timer
sudo systemctl start s3-backup.timer
```

Verify it is running:

```bash
systemctl list-timers | grep s3-backup
```

# Verifying Your Backups

List your backed up files:

```bash
aws s3 ls s3://your-homelab-backups/homelab-platform/ --recursive --human-readable
```

Check the backup log:

```bash
tail -f /var/log/s3-backup.log
```

# FAQ

## Why use the AWS CLI instead of rclone?

If you are only backing up to AWS S3, the AWS CLI is simpler - fewer dependencies, native AWS integration, and no extra configuration files. rclone makes more sense when you need to sync across multiple storage providers simultaneously.

## How do I restrict the IAM permissions?

Instead of full S3 access, create an IAM policy that only allows the specific actions your backup script needs:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::your-homelab-backups",
                "arn:aws:s3:::your-homelab-backups/*"
            ]
        }
    ]
}
```

## What does --delete do in sync?

The --delete flag removes files from S3 that no longer exist locally. Use it carefully - it is powerful for keeping S3 as an exact mirror but will permanently delete files that were removed from the source.
