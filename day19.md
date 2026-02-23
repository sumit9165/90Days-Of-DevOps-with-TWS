# Task 1: Log Rotation Script
---
```bash
#!/bin/bash

set -euo pipefail
# If total arguments not equal to 1 then script will exited.
if [[ $# -ne 1 ]]; then
    echo "Usage: $0 <log_directory>"
    exit 1
fi

LOG_DIR="$1"
# if directory does not exists then script will exited.
if [[ ! -d "$LOG_DIR" ]]; then
    echo "Error: Directory '$LOG_DIR' does not exist."
    exit 1
fi

# Get files first (store in arrays)
# log files find if files are older than 7 days.
mapfile -d '' LOG_FILES < <(find "$LOG_DIR" -type f -name "*.log" -mtime +7 -print0)
# gzip files find if files are older than 30 days.
mapfile -d '' GZ_FILES  < <(find "$LOG_DIR" -type f -name "*.gz"  -mtime +30 -print0)

# It will gzip the log files which are older  than 7 days.
if (( ${#LOG_FILES[@]} > 0 )); then
    printf '%s\0' "${LOG_FILES[@]}" | xargs -0 gzip
fi

# It will removing the gzip files which are older than 30 days.
if (( ${#GZ_FILES[@]} > 0 )); then
    printf '%s\0' "${GZ_FILES[@]}" | xargs -0 rm -f
fi

echo "Files compressed: ${#LOG_FILES[@]}"
echo "Files deleted: ${#GZ_FILES[@]}"
```
✔ Uses batch processing

Instead of:

gzip file1
gzip file2
gzip file3

It does:

gzip file1 file2 file3

Less process spawning → faster.

✔ No per-file loop

Reduces:

CPU overhead

Context switching

✔ Accurate counting via array length

No manual counters needed.

3️⃣ Summary of Important Commands
find

Search files with filters

Common flags:

-type f → files only

-name "*.log" → pattern match

-mtime +7 → older than 7 days

-print0 → null-delimited output

gzip

Compress file:

gzip file.log
rm -f

Force delete:

rm -f file.gz
xargs -0

Reads null-delimited input safely and batches execution.

mapfile

Reads command output into a Bash array.
----------
Task 2: Server Backup Script
----------
Below is a production-ready `backup.sh` script that fulfills all requirements with proper validation, error handling, and safe practices.

---

## ✅ `backup.sh`

```bash
#!/usr/bin/env bash

# backup.sh
# Usage: ./backup.sh <source_directory> <backup_destination>

set -euo pipefail

# ---- Argument Validation ----
if [[ $# -ne 2 ]]; then
    echo "Usage: $0 <source_directory> <backup_destination>"
    exit 1
fi

SOURCE="$1"
DEST="$2"

# ---- Validate Source ----
if [[ ! -d "$SOURCE" ]]; then
    echo "Error: Source directory '$SOURCE' does not exist."
    exit 1
fi

# ---- Validate/Create Destination ----
if [[ ! -d "$DEST" ]]; then
    echo "Destination directory does not exist. Creating it..."
    mkdir -p "$DEST"
fi

# ---- Create Timestamp ----
TIMESTAMP=$(date +"%Y-%m-%d")
ARCHIVE_NAME="backup-${TIMESTAMP}.tar.gz"
ARCHIVE_PATH="${DEST}/${ARCHIVE_NAME}"

# ---- Create Archive ----
tar -czf "$ARCHIVE_PATH" -C "$(dirname "$SOURCE")" "$(basename "$SOURCE")"

# ---- Verify Archive Creation ----
if [[ ! -f "$ARCHIVE_PATH" ]]; then
    echo "Error: Backup archive was not created."
    exit 1
fi

# ---- Get Archive Size ----
ARCHIVE_SIZE=$(du -h "$ARCHIVE_PATH" | cut -f1)

echo "Backup successful."
echo "Archive: $ARCHIVE_NAME"
echo "Size: $ARCHIVE_SIZE"

# ---- Delete Backups Older Than 14 Days ----
DELETED_COUNT=0

while IFS= read -r -d '' file; do
    rm -f "$file"
    ((DELETED_COUNT++))
done < <(find "$DEST" -type f -name "backup-*.tar.gz" -mtime +14 -print0)

echo "Old backups deleted: $DELETED_COUNT"

exit 0
```

---

# 🔎 What This Script Does (Step-by-Step)

### 1️⃣ Validates arguments

Requires exactly:

```
./backup.sh /source/path /backup/path
```

---

### 2️⃣ Verifies source exists

If not:

```
Error: Source directory does not exist.
```

Script exits immediately.

---

### 3️⃣ Creates timestamped archive

Format:

```
backup-YYYY-MM-DD.tar.gz
```

Example:

```
backup-2026-02-23.tar.gz
```

---

### 4️⃣ Uses proper tar structure

```
tar -czf archive.tar.gz -C parent_dir folder_name
```

This prevents storing full absolute paths inside the archive.

---

### 5️⃣ Verifies archive creation

Checks:

```bash
[[ -f "$ARCHIVE_PATH" ]]
```

---

### 6️⃣ Prints archive size

Uses:

```bash
du -h
```

---

### 7️⃣ Deletes backups older than 14 days

Uses:

```bash
find "$DEST" -name "backup-*.tar.gz" -mtime +14
```

Safe for filenames with spaces (`-print0`).

---
# Task 3: Crontab
---

# 1️⃣ Inspect Current Cron Jobs

```bash
crontab -l
```

If none exist:

```
no crontab for user
```

To edit:

```bash
crontab -e
```

---

# 2️⃣ Add the Entries (Safely)

Edit your crontab:

```bash
crontab -e
```

Add:

```bash
0 2 * * * /absolute/path/log_rotate.sh >> /var/log/log_rotate.log 2>&1
0 3 * * 0 /absolute/path/backup.sh >> /var/log/backup.log 2>&1
*/5 * * * * /absolute/path/health_check.sh >> /var/log/health_check.log 2>&1
```

⚠ Always use **absolute paths** (cron does not load your interactive shell environment).

---

# 3️⃣ Verify Scripts Are Executable

```bash
chmod +x /absolute/path/log_rotate.sh
chmod +x /absolute/path/backup.sh
chmod +x /absolute/path/health_check.sh
```

Test manually:

```bash
/absolute/path/log_rotate.sh
```

If it fails manually, it will fail in cron.

---

# 4️⃣ Validate Cron Is Running

On most Linux systems:

```bash
systemctl status cron
```

or

```bash
systemctl status crond
```

If inactive:

```bash
sudo systemctl start cron
```

---

# 5️⃣ Check Cron Logs (Debugging)

Ubuntu/Debian:

```bash
grep CRON /var/log/syslog
```

RHEL/CentOS:

```bash
grep CRON /var/log/cron
```

To watch live:

```bash
tail -f /var/log/syslog
```

---

# 6️⃣ Safe Testing Strategy (Recommended)

Instead of waiting until 2 AM:

Temporarily change timing to run in 1–2 minutes:

Example:

```bash
* * * * * /absolute/path/log_rotate.sh
```

Wait 1 minute and confirm:

* Log file updated
* Script executed correctly

Then revert back to the intended schedule.

---

# 7️⃣ Environment Pitfall (Common Production Issue)

Cron runs with a minimal environment.

To see what environment cron uses:

Add temporary debug job:

```bash
* * * * * env > /tmp/cron_env.txt
```

Wait one minute and inspect:

```bash
cat /tmp/cron_env.txt
```

If your script depends on:

* `$PATH`
* `$HOME`
* Python virtualenv
* NVM / Node
* Custom variables

You must define them explicitly inside the script.

---

# 8️⃣ Advanced Validation (Dry-Run Approach)

You can simulate execution environment:

```bash
env -i /bin/bash --noprofile --norc /absolute/path/script.sh
```

This mimics cron’s stripped-down environment.

---

If you tell me:

* Your Linux distribution
* Whether this is local, VM, or production server

I can tailor the debugging steps specifically to your environment.
--------------
# Task 4: Combine — Scheduled Maintenance Script
---

Below is a production-ready implementation.

---

# `maintenance.sh`

Assumptions:

* `log_rotate.sh` and `backup.sh` already exist
* They are executable
* You want centralized logging with timestamps
* Script runs with minimal cron environment

---

```bash
#!/bin/bash

# ===== Configuration =====
LOG_FILE="/var/log/maintenance.log"
LOG_ROTATE_SCRIPT="/absolute/path/log_rotate.sh"
BACKUP_SCRIPT="/absolute/path/backup.sh"

# ===== Timestamp Function =====
timestamp() {
    date "+%Y-%m-%d %H:%M:%S"
}

# ===== Logging Wrapper =====
log() {
    echo "$(timestamp) - $1" >> "$LOG_FILE"
}

# ===== Begin Maintenance =====
log "===== Maintenance Job Started ====="

# Run log rotation
if [ -x "$LOG_ROTATE_SCRIPT" ]; then
    log "Starting log rotation..."
    "$LOG_ROTATE_SCRIPT" >> "$LOG_FILE" 2>&1
    log "Log rotation completed with exit code $?"
else
    log "ERROR: Log rotation script not found or not executable."
fi

# Run backup
if [ -x "$BACKUP_SCRIPT" ]; then
    log "Starting backup..."
    "$BACKUP_SCRIPT" >> "$LOG_FILE" 2>&1
    log "Backup completed with exit code $?"
else
    log "ERROR: Backup script not found or not executable."
fi

log "===== Maintenance Job Finished ====="
echo "" >> "$LOG_FILE"
```

---

# Permissions

```bash
chmod +x /absolute/path/maintenance.sh
```

If `/var/log/maintenance.log` requires root access, run cron as root or adjust permissions.

---

# Cron Entry — Run Daily at 1 AM

```bash
0 1 * * * /absolute/path/maintenance.sh
```

If running as root:

```bash
sudo crontab -e
```
---
