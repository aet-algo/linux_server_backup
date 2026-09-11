# Ubuntu 26.04 LTS VPS: Automated Backup & Disaster Recovery Architecture

This repository contains a production-ready, automated backup and recovery pipeline for an Ubuntu 26.04 LTS Server. It is specifically optimized for quantitative finance environments running Python algorithmic trading scripts, Dockerized applications, and embedded databases (like DuckDB). 

The backup strategy utilizes a highly filtered `tar` snapshot to preserve filesystem metadata natively, stripping out heavy development artifacts (`.venv`, `node_modules`, Docker volumes) to save disk space, with an automated post-restoration script to rebuild the trading infrastructure.

---

## 1. The Automated Backup Script

This script creates a full system archive, handles standard live-system read warnings gracefully, and enforces a strict 7-day retention policy to prevent disk exhaustion.

**File Location:** `/usr/local/bin/system_backup.sh`

```bash
#!/bin/bash

# ==============================================================================
# Full System Backup & Retention Script (Development Optimized)
# ==============================================================================

# Configuration Variables
BACKUP_DEST="/var/backups/system_archives"
BACKUP_DATE=$(date +"%Y-%m-%d_%H-%M-%S")
ARCHIVE_NAME="ubuntu26_full_vps_$BACKUP_DATE.tar.gz"
LOG_FILE="/var/log/system_backup.log"
RETENTION_DAYS=7

# Define comprehensive exclusions: Virtual filesystems, package caches, Python/Dev artifacts
EXCLUDES=(
    "--exclude=$BACKUP_DEST"
    "--exclude=/proc"
    "--exclude=/tmp"
    "--exclude=/mnt"
    "--exclude=/dev"
    "--exclude=/sys"
    "--exclude=/run"
    "--exclude=/media"
    "--exclude=/lost+found"
    "--exclude=/var/tmp"
    "--exclude=/var/lib/docker"
    "--exclude=/var/cache/apt/archives"
    "--exclude=/home/*/.cache"
    "--exclude=/root/.cache"
    "--exclude=.Trash-*"
    "--exclude=.venv"
    "--exclude=__pycache__"
    "--exclude=.pytest_cache"
    "--exclude=.ipynb_checkpoints"
    "--exclude=*.pyc"
    "--exclude=*.pyo"
    "--exclude=*.pyd"
    "--exclude=*.log"
    "--exclude=.logs"
    "--exclude=*.bak"
    "--exclude=*.swp"
    "--exclude=*~"
    "--exclude=.git"
    "--exclude=node_modules"
)

# 1. Initialization & Logging
echo "==================================================" >> "$LOG_FILE"
echo "[$(date +'%Y-%m-%d %H:%M:%S')] Starting system backup..." >> "$LOG_FILE"

if [ ! -d "$BACKUP_DEST" ]; then
    mkdir -p "$BACKUP_DEST"
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] Created missing backup directory: $BACKUP_DEST" >> "$LOG_FILE"
fi

# 2. Execute Archive Process
echo "[$(date +'%Y-%m-%d %H:%M:%S')] Compressing file system..." >> "$LOG_FILE"
tar -cpzf "$BACKUP_DEST/$ARCHIVE_NAME" "${EXCLUDES[@]}" / >> "$LOG_FILE" 2>&1
TAR_EXIT_CODE=$?

if [ $TAR_EXIT_CODE -eq 0 ]; then
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] Backup completed successfully: $ARCHIVE_NAME" >> "$LOG_FILE"
elif [ $TAR_EXIT_CODE -eq 1 ]; then
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] Backup completed with non-fatal warnings (files changed during read)." >> "$LOG_FILE"
else
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] CRITICAL: Backup failed with tar exit code $TAR_EXIT_CODE." >> "$LOG_FILE"
    exit 1
fi

# 3. Clean Up Old Archives (7-Day Retention Policy)
echo "[$(date +'\%Y-\%m-\%d \%H:\%M:\%S')] Enforcing$RETENTION_DAYS-day retention policy..." >> "$LOG_FILE"
find "$BACKUP_DEST" -type f -name "ubuntu26_full_vps_*.tar.gz" -mtime +$RETENTION_DAYS -exec rm {} \; -exec echo "[$(date +'%Y-%m-%d %H:%M:%S')] Deleted old archive: {}" >> "$LOG_FILE" \;

echo "[$(date +'%Y-%m-%d %H:%M:%S')] Backup routine finished." >> "$LOG_FILE"
echo "==================================================" >> "$LOG_FILE"

exit 0
'''

###Deployment & Scheduling
1.  Apply execution permissions:
```bash
#!/bin/bash

sudo chmod +x /usr/local/bin/system_backup.sh
```

##2.  Schedule the script to run automatically every Sunday at 2:00 AM via the root crontab:
sudo crontab -e

##Add the following cron expression:
0 2 * * 0 /usr/local/bin/system_backup.sh
