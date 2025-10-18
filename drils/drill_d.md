# RHCSA DRILL D - Advanced System Administration
**Time Target: 60 minutes | Days 4, 8, 12, 16, 20, 24, 28**

---

## 🎯 SCENARIO 1: ARCHIVE, BACKUP & COMPRESSION (20 minutes)

### THE SITUATION:
Your organization needs efficient backup solutions using various compression methods. Master tar archives combined with gzip, bzip2, and xz compression for different backup scenarios.

**Requirements:**
- Create tar archives of system directories
- Compress archives using gzip, bzip2, and xz
- Extract and decompress archives
- Verify archive integrity
- Compare compression ratios
- Automate backup procedures

### YOUR TASKS:
1. Create tar archives (uncompressed)
2. Compress with gzip, bzip2, and xz
3. Create compressed archives in one step
4. Extract various archive types
5. List archive contents
6. Append to and update archives
7. Compare compression efficiency

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Create Test Environment:
```bash
# Create test directories and files
mkdir -p ~/backup-test/{documents,configs,logs}
echo "Important document" > ~/backup-test/documents/doc1.txt
echo "Another document" > ~/backup-test/documents/doc2.txt
echo "Configuration data" > ~/backup-test/configs/app.conf
echo "Database config" > ~/backup-test/configs/db.conf
for i in {1..10}; do
    echo "Log entry $i: $(date)" >> ~/backup-test/logs/app.log
done

# Create large file for testing compression
dd if=/dev/zero of=~/backup-test/largefile.dat bs=1M count=100
```

### Basic TAR Operations:

#### Create Uncompressed Archive:
```bash
# Create tar archive
tar -cvf backup.tar ~/backup-test/

# Options:
# -c: create
# -v: verbose
# -f: filename

# Without verbose
tar -cf backup.tar ~/backup-test/

# Verify archive was created
ls -lh backup.tar
```

#### List Archive Contents:
```bash
# List files in archive
tar -tvf backup.tar

# Options:
# -t: list contents
# -v: verbose
# -f: filename

# List specific file
tar -tvf backup.tar | grep "doc1.txt"

# Count files in archive
tar -tf backup.tar | wc -l
```

#### Extract Archive:
```bash
# Extract to current directory
tar -xvf backup.tar

# Extract to specific directory
tar -xvf backup.tar -C /tmp/

# Extract specific file
tar -xvf backup.tar home/user/backup-test/documents/doc1.txt

# Extract files matching pattern
tar -xvf backup.tar --wildcards '*.txt'
```

### GZIP Compression:

#### Compress with Gzip (.tar.gz or .tgz):
```bash
# Method 1: Create tar then compress
tar -cvf backup.tar ~/backup-test/
gzip backup.tar
# Result: backup.tar.gz

# Keep original tar file
gzip -k backup.tar

# Method 2: Create compressed archive in one step (PREFERRED)
tar -czvf backup.tar.gz ~/backup-test/

# Or using short extension
tar -czvf backup.tgz ~/backup-test/

# Options:
# -z: gzip compression
```

#### Decompress Gzip:
```bash
# Decompress .gz file
gunzip backup.tar.gz
# Result: backup.tar

# Keep compressed file
gunzip -k backup.tar.gz

# Extract tar.gz in one step (PREFERRED)
tar -xzvf backup.tar.gz

# Extract to specific directory
tar -xzvf backup.tar.gz -C /tmp/
```

#### View Gzip Archive Contents:
```bash
# List without extracting
tar -tzvf backup.tar.gz

# View specific file from archive
tar -xzvf backup.tar.gz --to-stdout home/user/backup-test/documents/doc1.txt
```

### BZIP2 Compression:

#### Compress with Bzip2 (.tar.bz2):
```bash
# Method 1: Create tar then compress
tar -cvf backup.tar ~/backup-test/
bzip2 backup.tar
# Result: backup.tar.bz2

# Keep original
bzip2 -k backup.tar

# Method 2: Create compressed archive in one step (PREFERRED)
tar -cjvf backup.tar.bz2 ~/backup-test/

# Options:
# -j: bzip2 compression

# Bzip2 has better compression than gzip but is slower
```

#### Decompress Bzip2:
```bash
# Decompress .bz2 file
bunzip2 backup.tar.bz2
# Result: backup.tar

# Keep compressed file
bunzip2 -k backup.tar.bz2

# Extract tar.bz2 in one step (PREFERRED)
tar -xjvf backup.tar.bz2

# Extract to specific directory
tar -xjvf backup.tar.bz2 -C /tmp/
```

### XZ Compression:

#### Compress with XZ (.tar.xz):
```bash
# Method 1: Create tar then compress
tar -cvf backup.tar ~/backup-test/
xz backup.tar
# Result: backup.tar.xz

# Keep original
xz -k backup.tar

# Method 2: Create compressed archive in one step (PREFERRED)
tar -cJvf backup.tar.xz ~/backup-test/

# Options:
# -J: xz compression (capital J!)

# XZ has best compression but is slowest
```

#### Decompress XZ:
```bash
# Decompress .xz file
unxz backup.tar.xz
# Result: backup.tar

# Keep compressed file
unxz -k backup.tar.xz

# Extract tar.xz in one step (PREFERRED)
tar -xJvf backup.tar.xz

# Extract to specific directory
tar -xJvf backup.tar.xz -C /tmp/
```

### Auto-detect Compression:

```bash
# Tar can auto-detect compression format
tar -xvf backup.tar.gz    # Auto-detects gzip
tar -xvf backup.tar.bz2   # Auto-detects bzip2
tar -xvf backup.tar.xz    # Auto-detects xz

# Using -a for auto-compression based on extension
tar -cavf backup.tar.gz ~/backup-test/   # Creates gzip
tar -cavf backup.tar.bz2 ~/backup-test/  # Creates bzip2
tar -cavf backup.tar.xz ~/backup-test/   # Creates xz
```

### Compare Compression Ratios:

```bash
# Create same archive with different compression
tar -cvf backup.tar ~/backup-test/
tar -czvf backup.tar.gz ~/backup-test/
tar -cjvf backup.tar.bz2 ~/backup-test/
tar -cJvf backup.tar.xz ~/backup-test/

# Compare sizes
ls -lh backup.tar*

# Example output:
# backup.tar     (100M - uncompressed)
# backup.tar.gz  (10M  - gzip, fast)
# backup.tar.bz2 (8M   - bzip2, medium)
# backup.tar.xz  (6M   - xz, best compression, slow)

# Detailed comparison
du -sh backup.tar*
```

### Advanced TAR Operations:

#### Append to Archive:
```bash
# Add files to existing archive (only uncompressed!)
echo "New file" > newfile.txt
tar -rvf backup.tar newfile.txt

# Cannot append to compressed archives directly
# Must decompress, append, then recompress
```

#### Update Archive:
```bash
# Update only newer files
tar -uvf backup.tar ~/backup-test/

# Only works with uncompressed archives
```

#### Exclude Files:
```bash
# Exclude specific files
tar -czvf backup.tar.gz --exclude='*.log' ~/backup-test/

# Exclude multiple patterns
tar -czvf backup.tar.gz \
  --exclude='*.log' \
  --exclude='*.tmp' \
  --exclude='cache/*' \
  ~/backup-test/

# Exclude file with list of patterns
cat > exclude.txt << EOF
*.log
*.tmp
cache/
.git/
EOF

tar -czvf backup.tar.gz --exclude-from=exclude.txt ~/backup-test/
```

#### Include Only Specific Files:
```bash
# Backup only .conf files
tar -czvf configs-only.tar.gz ~/backup-test/**/*.conf

# Using find with tar
find ~/backup-test -name "*.conf" | tar -czvf configs.tar.gz -T -
```

#### Preserve Permissions:
```bash
# Preserve permissions and ownership
tar -czvpf backup.tar.gz ~/backup-test/

# -p: preserve permissions

# Extract preserving permissions
tar -xzvpf backup.tar.gz
```

#### Differential/Incremental Backup:
```bash
# Full backup with snapshot
tar -czvf full-backup.tar.gz -g snapshot.file ~/backup-test/

# Incremental backup (only changed files)
tar -czvf incremental-backup.tar.gz -g snapshot.file ~/backup-test/

# The snapshot.file tracks what was backed up
```

### Practical Backup Scenarios:

#### System Configuration Backup:
```bash
# Backup /etc directory
sudo tar -czvf etc-backup-$(date +%Y%m%d).tar.gz /etc/

# Backup with exclusions
sudo tar -czvf etc-backup.tar.gz \
  --exclude='/etc/shadow*' \
  --exclude='/etc/ssl/private' \
  /etc/
```

#### User Home Directory Backup:
```bash
# Backup home directory
tar -czvf home-backup-$(date +%Y%m%d).tar.gz \
  --exclude='Downloads' \
  --exclude='.cache' \
  --exclude='.local/share/Trash' \
  ~/
```

#### Log Files Backup:
```bash
# Backup logs (high compression)
sudo tar -cJvf logs-backup-$(date +%Y%m%d).tar.xz /var/log/

# Exclude old compressed logs
sudo tar -czvf logs-backup.tar.gz \
  --exclude='*.gz' \
  --exclude='*.1' \
  /var/log/
```

#### Database Backup:
```bash
# Backup MySQL data directory
sudo tar -czvf mysql-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/mysql/

# With exclusions
sudo tar -czvf mysql-backup.tar.gz \
  --exclude='*.log' \
  /var/lib/mysql/
```

#### Website Backup:
```bash
# Backup web root
sudo tar -czvf website-backup-$(date +%Y%m%d).tar.gz \
  --exclude='cache/*' \
  --exclude='tmp/*' \
  /var/www/html/
```

### Compression Comparison Script:

```bash
# Create comparison script
cat > compare-compression.sh << 'EOF'
#!/bin/bash

SOURCE="$1"
if [ -z "$SOURCE" ]; then
    echo "Usage: $0 <directory>"
    exit 1
fi

echo "Comparing compression methods for: $SOURCE"
echo "============================================"

# Uncompressed
time tar -cf /tmp/test.tar $SOURCE 2>/dev/null
SIZE_TAR=$(du -h /tmp/test.tar | cut -f1)
echo "TAR (uncompressed): $SIZE_TAR"
rm /tmp/test.tar

# Gzip
time tar -czf /tmp/test.tar.gz $SOURCE 2>/dev/null
SIZE_GZ=$(du -h /tmp/test.tar.gz | cut -f1)
echo "GZIP: $SIZE_GZ"
rm /tmp/test.tar.gz

# Bzip2
time tar -cjf /tmp/test.tar.bz2 $SOURCE 2>/dev/null
SIZE_BZ2=$(du -h /tmp/test.tar.bz2 | cut -f1)
echo "BZIP2: $SIZE_BZ2"
rm /tmp/test.tar.bz2

# XZ
time tar -cJf /tmp/test.tar.xz $SOURCE 2>/dev/null
SIZE_XZ=$(du -h /tmp/test.tar.xz | cut -f1)
echo "XZ: $SIZE_XZ"
rm /tmp/test.tar.xz

echo "============================================"
echo "Best compression: XZ (slowest)"
echo "Fastest: GZIP"
echo "Balanced: BZIP2"
EOF

chmod +x compare-compression.sh
./compare-compression.sh ~/backup-test
```

### Automated Backup Script:

```bash
# Create automated backup script
cat > daily-backup.sh << 'EOF'
#!/bin/bash

# Configuration
BACKUP_SOURCE="/home/data"
BACKUP_DEST="/backups"
DATE=$(date +%Y%m%d)
RETENTION_DAYS=7

# Create backup directory
mkdir -p $BACKUP_DEST

# Create backup
echo "Starting backup at $(date)"
tar -czvf $BACKUP_DEST/backup-$DATE.tar.gz \
  --exclude='*.tmp' \
  --exclude='.cache' \
  $BACKUP_SOURCE

# Check if backup succeeded
if [ $? -eq 0 ]; then
    echo "Backup completed successfully"
    
    # Remove old backups
    find $BACKUP_DEST -name "backup-*.tar.gz" -mtime +$RETENTION_DAYS -delete
    echo "Old backups removed (older than $RETENTION_DAYS days)"
else
    echo "Backup failed!"
    exit 1
fi

echo "Backup finished at $(date)"
EOF

chmod +x daily-backup.sh
```

### Verification and Testing:

```bash
# Create test archive
tar -czvf test.tar.gz ~/backup-test/

# Verify archive integrity
tar -tzvf test.tar.gz > /dev/null
echo $?  # Should be 0 if OK

# Extract to test directory
mkdir /tmp/restore-test
tar -xzvf test.tar.gz -C /tmp/restore-test/

# Compare original and extracted
diff -r ~/backup-test/ /tmp/restore-test/home/$(whoami)/backup-test/

# Test each compression type
echo "Testing GZIP..."
tar -czvf test.tar.gz ~/backup-test/ && tar -tzvf test.tar.gz > /dev/null && echo "GZIP: OK"

echo "Testing BZIP2..."
tar -cjvf test.tar.bz2 ~/backup-test/ && tar -tjvf test.tar.bz2 > /dev/null && echo "BZIP2: OK"

echo "Testing XZ..."
tar -cJvf test.tar.xz ~/backup-test/ && tar -tJvf test.tar.xz > /dev/null && echo "XZ: OK"

# Clean up
rm -f test.tar.*
rm -rf /tmp/restore-test
```

### Quick Reference:

```bash
# Create compressed archives
tar -czvf file.tar.gz directory/   # gzip (fast)
tar -cjvf file.tar.bz2 directory/  # bzip2 (better compression)
tar -cJvf file.tar.xz directory/   # xz (best compression)

# Extract
tar -xzvf file.tar.gz
tar -xjvf file.tar.bz2
tar -xJvf file.tar.xz

# Or use auto-detect
tar -xvf file.tar.*

# List contents
tar -tzvf file.tar.gz

# Create with exclusions
tar -czvf backup.tar.gz --exclude='*.log' directory/
```

</details>

---

## 🎯 SCENARIO 2: ACCESS CONTROL LISTS (ACL) (15 minutes)

### THE SITUATION:
Your organization has complex permission requirements. Standard Unix permissions (ugo/rwx) aren't sufficient. Multiple users need different access levels to the same files without changing ownership or group membership.

**Requirements:**
- Development team needs shared access to `/projects/webapp`
- User `alice` needs read/write to specific files in `/projects/webapp`
- User `bob` needs read-only access to specific files
- Group `developers` needs full access to `/projects/data`
- Prevent user `charlie` from accessing certain files entirely

### YOUR TASKS:
1. Create test directory structure and users
2. Set ACLs for specific user permissions
3. Set ACLs for group permissions
4. Remove user access using ACL
5. Apply ACLs recursively
6. Set default ACLs for new files
7. View and verify ACL settings

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Create Test Environment:
```bash
# Create users
sudo useradd alice
sudo useradd bob
sudo useradd charlie
echo "password123" | sudo passwd --stdin alice
echo "password123" | sudo passwd --stdin bob
echo "password123" | sudo passwd --stdin charlie

# Create group
sudo groupadd developers
sudo usermod -aG developers alice
sudo usermod -aG developers bob

# Create directory structure
sudo mkdir -p /projects/webapp
sudo mkdir -p /projects/data
sudo mkdir -p /projects/logs

# Create test files
echo "Web Application Code" | sudo tee /projects/webapp/app.py
echo "Configuration File" | sudo tee /projects/webapp/config.ini
echo "Development Data" | sudo tee /projects/data/devdata.txt
echo "Log entries" | sudo tee /projects/logs/app.log
```

### Set File Ownership:
```bash
# Set ownership
sudo chown root:root /projects/webapp/app.py
sudo chown root:root /projects/webapp/config.ini
sudo chown root:developers /projects/data/devdata.txt

# Base permissions
sudo chmod 640 /projects/webapp/app.py
sudo chmod 640 /projects/webapp/config.ini
sudo chmod 660 /projects/data/devdata.txt

# Verify
ls -l /projects/webapp/
ls -l /projects/data/
```

### Grant User-Specific Permissions with ACL:
```bash
# Give alice read/write access to app.py
sudo setfacl --modify user:alice:rw /projects/webapp/app.py

# Give bob read-only access to app.py
sudo setfacl --modify user:bob:r /projects/webapp/config.ini

# Verify - notice the + sign indicating ACL
ls -l /projects/webapp/app.py
# Output: -rw-rw----+ ...

# View detailed ACL
getfacl /projects/webapp/app.py
```

### Grant Group Permissions with ACL:
```bash
# Give developers group full access to data directory
sudo setfacl --modify group:developers:rwx /projects/data/devdata.txt

# Verify
getfacl /projects/data/devdata.txt
```

### Remove User Access Completely:
```bash
# Deny charlie all access to config.ini
sudo setfacl --modify user:charlie:--- /projects/webapp/config.ini

# Verify
getfacl /projects/webapp/config.ini

# Test as charlie (should fail)
su - charlie
cat /projects/webapp/config.ini
# Output: Permission denied
exit
```

### Apply ACLs Recursively:
```bash
# Create subdirectories with files
sudo mkdir -p /projects/webapp/modules
echo "Module code" | sudo tee /projects/webapp/modules/auth.py
echo "Module code" | sudo tee /projects/webapp/modules/database.py

# Apply ACL recursively
sudo setfacl --recursive --modify user:alice:rwx /projects/webapp/

# Verify all files
getfacl /projects/webapp/app.py
getfacl /projects/webapp/modules/auth.py
```

### Set Default ACLs for New Files:
```bash
# Set default ACL on directory
# All NEW files created here will inherit these ACLs
sudo setfacl --modify default:user:alice:rw /projects/webapp/
sudo setfacl --modify default:group:developers:rw /projects/webapp/

# View default ACLs
getfacl /projects/webapp/

# Test by creating a new file
sudo touch /projects/webapp/newfile.txt

# Check if new file inherited ACL
getfacl /projects/webapp/newfile.txt
# Should show alice:rw- as inherited ACL
```

### Modify ACL Mask:
```bash
# The mask limits maximum permissions for named users/groups
# Set mask to read-only
sudo setfacl --modify mask:r /projects/webapp/app.py

# Even if alice has rw, mask limits to r
getfacl /projects/webapp/app.py
```

### Remove ACL Entries:
```bash
# Remove specific user ACL
sudo setfacl --remove user:bob /projects/webapp/config.ini

# Remove specific group ACL
sudo setfacl --remove group:developers /projects/data/devdata.txt

# Remove ALL ACLs from a file (revert to standard permissions)
sudo setfacl --remove-all /projects/webapp/app.py

# Remove default ACLs
sudo setfacl --remove-default /projects/webapp/
```

### Copy ACLs Between Files:
```bash
# Copy ACL from one file to another
getfacl /projects/webapp/app.py | sudo setfacl --set-file=- /projects/webapp/config.ini
```

### Verification:
```bash
# Check all files with ACLs in directory
ls -l /projects/webapp/
# Files with + have ACLs

# View all ACLs
getfacl -R /projects/webapp/

# Test as alice
su - alice
cat /projects/webapp/app.py          # Should work
echo "test" >> /projects/webapp/app.py  # Should work or fail based on ACL
exit

# Test as bob
su - bob
cat /projects/webapp/config.ini       # Should work
echo "test" >> /projects/webapp/config.ini  # Should fail
exit
```

### Common ACL Scenarios:

```bash
# Scenario 1: Shared project folder
sudo mkdir /shared_project
sudo setfacl -m u:alice:rwx /shared_project
sudo setfacl -m u:bob:rx /shared_project
sudo setfacl -m d:u:alice:rwx /shared_project  # default for new files
sudo setfacl -m d:u:bob:rx /shared_project

# Scenario 2: Log file access
sudo setfacl -m u:alice:r /var/log/application.log
sudo setfacl -m g:developers:r /var/log/application.log

# Scenario 3: Backup directory with limited access
sudo setfacl -R -m u:backup_user:rx /data/backups
sudo setfacl -m u:backup_user:rwx /data/backups  # can write to dir
```

### Final Verification:
```bash
# List all files with ACLs
find /projects -type f -exec ls -l {} \; | grep "+"

# Comprehensive ACL report
getfacl -R /projects/ > /tmp/acl_report.txt
cat /tmp/acl_report.txt
```

</details>

---

## 🎯 SCENARIO 3: SYSTEM LOGGING & JOURNALCTL (10 minutes)

### THE SITUATION:
Troubleshoot system issues by analyzing logs. Configure persistent journal storage and master log filtering techniques.

### YOUR TASKS:
1. View system logs with journalctl
2. Filter logs by time, service, priority
3. Configure persistent journal storage
4. Export and analyze logs
5. Monitor logs in real-time

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Basic Journalctl Usage:
```bash
# View all logs
sudo journalctl

# Most recent logs first
sudo journalctl -r

# Last 50 lines (like tail)
sudo journalctl -n 50

# Follow logs in real-time (like tail -f)
sudo journalctl -f

# Show only last boot
sudo journalctl -b

# Show previous boot
sudo journalctl -b -1

# List all boots
sudo journalctl --list-boots
```

### Filter by Time:
```bash
# Today's logs
sudo journalctl --since today

# Yesterday's logs
sudo journalctl --since yesterday --until today

# Specific date
sudo journalctl --since "2025-01-15"

# Date and time
sudo journalctl --since "2025-01-15 10:00:00" --until "2025-01-15 18:00:00"

# Last hour
sudo journalctl --since "1 hour ago"

# Last 30 minutes
sudo journalctl --since "30 min ago"

# Last 2 days
sudo journalctl --since "2 days ago"
```

### Filter by Service/Unit:
```bash
# Specific service
sudo journalctl -u sshd.service

# Multiple services
sudo journalctl -u sshd.service -u httpd.service

# Follow specific service
sudo journalctl -u sshd.service -f

# Kernel messages only
sudo journalctl -k

# Since last boot for specific service
sudo journalctl -b -u sshd.service
```

### Filter by Priority:
```bash
# Priority levels: emerg(0), alert(1), crit(2), err(3), warning(4), notice(5), info(6), debug(7)

# Show only errors and above
sudo journalctl -p err

# Show warnings and above
sudo journalctl -p warning

# Specific priority
sudo journalctl -p 3  # err

# Range
sudo journalctl -p err..emerg

# Errors from specific service
sudo journalctl -u sshd.service -p err
```

### Filter by Process/User:
```bash
# By PID
sudo journalctl _PID=1234

# By UID
sudo journalctl _UID=1000

# By GID
sudo journalctl _GID=1000

# By executable
sudo journalctl /usr/bin/sudo

# By command
sudo journalctl -u sshd.service _COMM=sshd
```

### Output Formats:
```bash
# Verbose (show all fields)
sudo journalctl -o verbose

# JSON format
sudo journalctl -o json

# JSON pretty
sudo journalctl -o json-pretty

# Short format (default)
sudo journalctl -o short

# Export format (for backup)
sudo journalctl -o export > journal-backup.export

# Cat format (no metadata)
sudo journalctl -o cat
```

### Combine Filters:
```bash
# Service + time + priority
sudo journalctl -u httpd.service --since today -p err

# Multiple services + last hour
sudo journalctl -u sshd.service -u httpd.service --since "1 hour ago"

# Boot + service
sudo journalctl -b -u NetworkManager.service

# Time range + follow
sudo journalctl --since "10 min ago" -f
```

### Configure Persistent Journal Storage:
```bash
# Check current storage
sudo journalctl --disk-usage

# Check if persistent
ls -la /var/log/journal/

# If directory doesn't exist, create it
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal

# Configure persistence
sudo vi /etc/systemd/journald.conf
```

Edit:
```
[Journal]
Storage=persistent
SystemMaxUse=500M
SystemMaxFileSize=100M
RuntimeMaxUse=100M
```

```bash
# Restart journald
sudo systemctl restart systemd-journald

# Verify
sudo journalctl --verify
ls -lh /var/log/journal/
```

### Journal Maintenance:
```bash
# Check disk usage
sudo journalctl --disk-usage

# Vacuum by size (keep only 500M)
sudo journalctl --vacuum-size=500M

# Vacuum by time (keep only 7 days)
sudo journalctl --vacuum-time=7d

# Vacuum by files (keep only 5 files)
sudo journalctl --vacuum-files=5

# Rotate journals
sudo journalctl --rotate

# Verify journal integrity
sudo journalctl --verify
```

### Export and Analyze:
```bash
# Export to file
sudo journalctl --since today > /tmp/today-logs.txt

# Export JSON
sudo journalctl -o json --since today > /tmp/today-logs.json

# Export specific service
sudo journalctl -u sshd.service --since yesterday > /tmp/sshd-logs.txt

# Count log entries
sudo journalctl --since today | wc -l

# Find errors in exports
grep -i "error" /tmp/today-logs.txt
grep -i "failed" /tmp/today-logs.txt
```

### Real-Time Monitoring:
```bash
# Follow all logs
sudo journalctl -f

# Follow with priority
sudo journalctl -f -p warning

# Follow specific service
sudo journalctl -f -u httpd.service

# Follow kernel messages
sudo journalctl -f -k

# Follow multiple services
sudo journalctl -f -u sshd.service -u httpd.service
```

### Practical Troubleshooting Examples:

```bash
# Example 1: SSH login failures
sudo journalctl -u sshd.service | grep "Failed password"

# Example 2: System boot issues
sudo journalctl -b -p err

# Example 3: Service start failures
sudo journalctl -u httpd.service --since "1 hour ago"

# Example 4: Find when system rebooted
sudo journalctl --list-boots

# Example 5: All logs from last boot
sudo journalctl -b -1

# Example 6: Kernel errors
sudo journalctl -k -p err

# Example 7: SELinux denials
sudo journalctl -t setroubleshoot

# Example 8: Failed systemd units
sudo journalctl -p err -u "*.service"
```

### Advanced Queries:
```bash
# Logs from specific boot with time range
sudo journalctl -b --since "10:00" --until "11:00"

# Count errors per service
sudo journalctl -p err -o json | jq -r '._SYSTEMD_UNIT' | sort | uniq -c

# Show log entry fields
sudo journalctl -o verbose -n 1

# Search for specific message
sudo journalctl | grep -i "authentication failure"

# Failed services in last boot
sudo journalctl -b -p err -u "*.service" | grep "Failed"
```

### Traditional Log Files:
```bash
# Most logs still in /var/log/
ls -lh /var/log/

# Important log files
sudo tail -f /var/log/messages    # General system log
sudo tail -f /var/log/secure      # Authentication log
sudo tail -f /var/log/audit/audit.log  # Audit log
sudo tail -f /var/log/maillog    # Mail server log
sudo tail -f /var/log/cron       # Cron job log

# View with journalctl
sudo journalctl -f /var/log/messages
```

### Verification:
```bash
# Check journal status
sudo systemctl status systemd-journald

# Verify persistent storage
ls -lh /var/log/journal/

# Check disk usage
sudo journalctl --disk-usage

# Test log entry
logger "Test log entry"
sudo journalctl -n 5

# Test priority filtering
sudo journalctl -p warning --since "1 min ago"

# Verify specific service logs
sudo journalctl -u sshd.service -n 20
```

</details>

---

## 🎯 SCENARIO 5: GREP WITH REGULAR EXPRESSIONS (10 minutes)

### THE SITUATION:
Master text searching and pattern matching for log analysis, configuration file parsing, and system troubleshooting.

### YOUR TASKS:
1. Basic grep usage and options
2. Use regular expressions for pattern matching
3. Extended regular expressions (egrep)
4. Practical log analysis scenarios
5. Combine grep with other commands

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Create Test Files:
```bash
# Create test log file
cat > /tmp/test.log << 'EOF'
2025-01-15 10:23:45 ERROR Failed to connect to database
2025-01-15 10:24:12 INFO User alice logged in
2025-01-15 10:25:33 WARNING Disk space low on /dev/sda1
2025-01-15 10:26:01 ERROR Connection timeout: 192.168.1.100
2025-01-15 10:27:15 INFO User bob logged in from 10.0.0.5
2025-01-15 10:28:22 CRITICAL System temperature: 85C
2025-01-15 10:29:45 ERROR Authentication failed for user charlie
2025-01-15 10:30:11 INFO Backup completed successfully
2025-01-15 10:31:00 WARNING CPU usage at 95%
EOF

# Create test config file
cat > /tmp/test.conf << 'EOF'
# Network Configuration
interface=eth0
ip_address=192.168.1.50
netmask=255.255.255.0
gateway=192.168.1.1

# Email settings
smtp_server=mail.example.com
smtp_port=587
admin_email=admin@example.com

# Database
db_host=localhost
db_port=3306
db_name=myapp
db_user=appuser
EOF
```

### Basic Grep Usage:
```bash
# Search for exact string
grep "ERROR" /tmp/test.log

# Case-insensitive search
grep -i "error" /tmp/test.log

# Show line numbers
grep -n "ERROR" /tmp/test.log

# Count matches
grep -c "ERROR" /tmp/test.log

# Show lines NOT matching
grep -v "INFO" /tmp/test.log

# Multiple files
grep "ERROR" /tmp/test.log /var/log/messages

# Recursive search in directory
grep -r "failed" /var/log/
```

### Extended Options:
```bash
# Show N lines after match
grep -A 2 "ERROR" /tmp/test.log

# Show N lines before match
grep -B 2 "ERROR" /tmp/test.log

# Show N lines before and after (context)
grep -C 2 "ERROR" /tmp/test.log

# Only show matching part
grep -o "192\.[0-9]*\.[0-9]*\.[0-9]*" /tmp/test.log

# Show filename only
grep -l "ERROR" /tmp/*.log

# Show files NOT containing pattern
grep -L "SUCCESS" /tmp/*.log

# Suppress error messages
grep -s "pattern" /nonexistent/file

# Quiet mode (just exit code)
grep -q "ERROR" /tmp/test.log && echo "Errors found"
```

### Basic Regular Expressions:
```bash
# Anchors
grep "^2025" /tmp/test.log           # Lines starting with 2025
grep "successfully$" /tmp/test.log    # Lines ending with successfully

# Dot (any single character)
grep "19." /tmp/test.log              # 192, 193, 195, etc.

# Character classes
grep "[Ee]rror" /tmp/test.log         # Error or error
grep "[0-9]" /tmp/test.log            # Any digit
grep "[a-z]" /tmp/test.log            # Any lowercase letter
grep "[^0-9]" /tmp/test.log           # NOT a digit

# Ranges
grep "[0-9][0-9]:[0-9][0-9]" /tmp/test.log  # Time pattern

# Repetition
grep "o*" /tmp/test.log               # Zero or more 'o'
grep "er*" /tmp/test.log              # 'e' followed by zero or more 'r'
```

### Extended Regular Expressions (use -E or egrep):
```bash
# + (one or more)
grep -E "[0-9]+" /tmp/test.log        # One or more digits

# ? (zero or one)
grep -E "colou?r" /tmp/test.log       # color or colour

# | (OR)
grep -E "ERROR|CRITICAL" /tmp/test.log

# () (grouping)
grep -E "(alice|bob|charlie)" /tmp/test.log

# {n} (exactly n)
grep -E "[0-9]{3}" /tmp/test.log      # Exactly 3 digits

# {n,} (n or more)
grep -E "[0-9]{2,}" /tmp/test.log     # 2 or more digits

# {n,m} (between n and m)
grep -E "[0-9]{1,3}" /tmp/test.log    # 1 to 3 digits

# Word boundaries
grep -E "\broot\b" /etc/passwd         # Exact word "root"
```

### Practical IP Address Matching:
```bash
# Simple IP pattern (not perfect)
grep -E "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" /tmp/test.log

# Better IP pattern
grep -E "([0-9]{1,3}\.){3}[0-9]{1,3}" /tmp/test.log

# Extract only IPs
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" /tmp/test.log
```

### Email Address Matching:
```bash
# Basic email pattern
grep -E "[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" /tmp/test.conf

# Extract emails
grep -Eo "[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" /tmp/test.conf
```

### Log Analysis Examples:

```bash
# Find all errors and warnings
grep -E "ERROR|WARNING|CRITICAL" /tmp/test.log

# Extract timestamps
grep -Eo "[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}" /tmp/test.log

# Find failed logins
grep "failed" /var/log/secure

# Count failed SSH attempts
grep "Failed password" /var/log/secure | wc -l

# Extract unique IPs from logs
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" /tmp/test.log | sort -u

# Find lines with temperature > 80
grep -E "temperature.*[8-9][0-9]C" /tmp/test.log
```

### Searching Configuration Files:
```bash
# Find all non-comment, non-empty lines
grep -Ev "^#|^$" /tmp/test.conf

# Extract IP addresses from configs
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" /tmp/test.conf

# Find specific settings
grep "^db_" /tmp/test.conf

# Case-insensitive config search
grep -i "smtp" /tmp/test.conf
```

### Combining with Other Commands:

```bash
# Grep with find
find /var/log -name "*.log" -exec grep -l "ERROR" {} \;

# Grep with ps
ps aux | grep -v grep | grep httpd

# Grep with df
df -h | grep -E "8[0-9]%|9[0-9]%"  # Find >80% full

# Grep with netstat/ss
ss -tuln | grep ":80\|:443"

# Grep with awk
grep "ERROR" /tmp/test.log | awk '{print $1, $2, $4}'

# Grep with sed
grep "alice" /tmp/test.log | sed 's/alice/ALICE/g'

# Multiple greps (pipeline)
grep "ERROR" /tmp/test.log | grep "database"

# Exclude multiple patterns
grep -E "ERROR|WARNING" /tmp/test.log | grep -v "Disk space"
```

### Advanced Patterns:

```bash
# MAC address
grep -Eo "([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}" /path/to/file

# URL pattern
grep -Eo "https?://[a-zA-Z0-9./?=_-]*" /path/to/file

# Phone number (US format)
grep -Eo "\(?[0-9]{3}\)?[-. ]?[0-9]{3}[-. ]?[0-9]{4}" /path/to/file

# Date (YYYY-MM-DD)
grep -Eo "[0-9]{4}-[0-9]{2}-[0-9]{2}" /tmp/test.log

# Time (HH:MM:SS)
grep -Eo "[0-9]{2}:[0-9]{2}:[0-9]{2}" /tmp/test.log
```

### Real-World Scenarios:

```bash
# Scenario 1: Find failed SSH logins with IPs
grep "Failed password" /var/log/secure | grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" | sort | uniq -c | sort -rn

# Scenario 2: Find errors in last hour
journalctl --since "1 hour ago" | grep -i error

# Scenario 3: Check which users logged in today
grep "session opened" /var/log/secure | grep $(date +%b\ %d)

# Scenario 4: Find all listening ports
ss -tuln | grep LISTEN | grep -Eo ":[0-9]+" | sort -u

# Scenario 5: Extract all email addresses from files
grep -rEho "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b" /etc/ | sort -u

# Scenario 6: Find configuration files with passwords
grep -r "password=" /etc/*.conf

# Scenario 7: Check for specific error codes
grep -E "error.*[45][0-9]{2}" /var/log/httpd/error_log
```

### Performance Tips:

```bash
# Use -F for fixed strings (faster)
grep -F "exact.string" large_file.log

# Use -m to stop after N matches
grep -m 5 "ERROR" /var/log/messages

# Exclude binary files
grep -I "pattern" *

# Use multiple patterns file
cat > /tmp/patterns.txt << EOF
ERROR
CRITICAL
WARNING
EOF
grep -f /tmp/patterns.txt /tmp/test.log
```

### Color Output:
```bash
# Enable colors
grep --color=auto "ERROR" /tmp/test.log

# Set alias for colored grep
alias grep='grep --color=auto'

# Force color even when piping
grep --color=always "ERROR" /tmp/test.log | less -R
```

### Verification:
```bash
# Test basic search
grep "ERROR" /tmp/test.log

# Test regex
grep -E "ERROR|WARNING" /tmp/test.log

# Test case-insensitive
grep -i "error" /tmp/test.log

# Test IP extraction
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" /tmp/test.log

# Test context
grep -C 1 "CRITICAL" /tmp/test.log

# Test inverted match
grep -v "INFO" /tmp/test.log | grep -v "^$"

# Verify count
grep -c "ERROR" /tmp/test.log
echo "Should be 3"
```

</details>

---

## 🎯 SCENARIO 6: REMOTE FILE TRANSFER (SCP & RSYNC) (10 minutes)

### THE SITUATION:
Transfer files securely between systems for backups, deployments, and system administration tasks. Use both scp for simple transfers and rsync for efficient synchronization.

### YOUR TASKS:
1. Transfer files with scp
2. Transfer directories with scp
3. Use rsync for efficient sync
4. Sync with compression and excludes
5. Practice common transfer scenarios

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### SCP Basics:

#### Copy File to Remote:
```bash
# Basic syntax: scp source destination

# Copy local file to remote
scp /path/to/local/file.txt user@remote:/path/to/remote/

# Specify port
scp -P 2222 file.txt user@remote:/path/

# Preserve permissions and timestamps
scp -p file.txt user@remote:/path/

# Verbose output
scp -v file.txt user@remote:/path/
```

#### Copy File from Remote:
```bash
# Copy from remote to local
scp user@remote:/path/to/remote/file.txt /local/path/

# Copy to current directory
scp user@remote:/path/to/file.txt .

# Multiple files
scp user@remote:/path/to/file{1,2,3}.txt /local/path/
```

#### Copy Directories:
```bash
# Copy directory recursively
scp -r /local/directory user@remote:/remote/path/

# Copy from remote
scp -r user@remote:/remote/directory /local/path/

# Preserve permissions
scp -rp /local/directory user@remote:/remote/path/
```

#### Advanced SCP Options:
```bash
# Limit bandwidth (in Kbit/s)
scp -l 1000 large_file.tar.gz user@remote:/path/

# Use compression
scp -C large_file.tar.gz user@remote:/path/

# Use specific SSH key
scp -i ~/.ssh/id_rsa file.txt user@remote:/path/

# Copy between two remote hosts
scp user1@host1:/path/file.txt user2@host2:/path/
```

### RSYNC Basics:

#### Basic Sync:
```bash
# Basic syntax: rsync [options] source destination

# Sync local to remote
rsync -av /local/dir/ user@remote:/remote/dir/

# Key options:
# -a (archive): preserves permissions, ownership, timestamps, recursive
# -v (verbose): detailed output
# -z (compress): compress during transfer
# -h (human-readable): human-readable numbers

# Sync from remote to local
rsync -av user@remote:/remote/dir/ /local/dir/
```

#### Dry Run (Test First):
```bash
# Show what would be transferred without doing it
rsync -avn /local/dir/ user@remote:/remote/dir/

# or
rsync -av --dry-run /local/dir/ user@remote:/remote/dir/
```

#### Delete Files on Destination:
```bash
# Delete files on destination that don't exist in source
rsync -av --delete /local/dir/ user@remote:/remote/dir/

# CAREFUL: This makes destination exactly match source
```

#### Progress and Stats:
```bash
# Show progress during transfer
rsync -av --progress /local/dir/ user@remote:/remote/dir/

# Show summary stats
rsync -av --stats /local/dir/ user@remote:/remote/dir/

# Both
rsync -av --progress --stats /local/dir/ user@remote:/remote/dir/
```

#### Exclude Files/Directories:
```bash
# Exclude specific files
rsync -av --exclude='*.log' /local/dir/ user@remote:/remote/dir/

# Exclude multiple patterns
rsync -av --exclude='*.log' --exclude='*.tmp' /local/dir/ user@remote:/remote/dir/

# Exclude directory
rsync -av --exclude='cache/' /local/dir/ user@remote:/remote/dir/

# Use exclude file
cat > /tmp/exclude.txt << EOF
*.log
*.tmp
cache/
.git/
EOF

rsync -av --exclude-from=/tmp/exclude.txt /local/dir/ user@remote:/remote/dir/
```

#### Include Only Specific Files:
```bash
# Include only certain files
rsync -av --include='*.txt' --exclude='*' /local/dir/ user@remote:/remote/dir/

# Include specific directory structure
rsync -av --include='*/' --include='*.conf' --exclude='*' /etc/ /backup/etc/
```

#### Bandwidth Limiting:
```bash
# Limit bandwidth (in KB/s)
rsync -av --bwlimit=1000 /local/dir/ user@remote:/remote/dir/
```

#### SSH Options:
```bash
# Specify SSH port
rsync -av -e "ssh -p 2222" /local/dir/ user@remote:/remote/dir/

# Use specific SSH key
rsync -av -e "ssh -i ~/.ssh/id_rsa" /local/dir/ user@remote:/remote/dir/

# Combine SSH options
rsync -av -e "ssh -p 2222 -i ~/.ssh/id_rsa" /local/dir/ user@remote:/remote/dir/
```

#### Partial Transfers:
```bash
# Resume partial transfers
rsync -av --partial /local/dir/ user@remote:/remote/dir/

# Keep partial files for resume
rsync -av --partial-dir=/tmp/rsync-partial /local/dir/ user@remote:/remote/dir/

# Resume interrupted transfer (combination)
rsync -avz --partial --progress /local/dir/ user@remote:/remote/dir/
```

### Practical Examples:

#### Example 1: Website Deployment
```bash
# Deploy website with specific exclusions
rsync -avz --delete \
  --exclude='.git/' \
  --exclude='node_modules/' \
  --exclude='*.log' \
  /local/website/ user@webserver:/var/www/html/
```

#### Example 2: Database Backup
```bash
# Backup database dumps
rsync -avz --progress \
  /var/backups/mysql/ \
  user@backup-server:/backups/mysql/$(date +%Y%m%d)/
```

#### Example 3: Home Directory Backup
```bash
# Backup home directory excluding cache
rsync -av --delete \
  --exclude='.cache/' \
  --exclude='Downloads/' \
  --exclude='.local/share/Trash/' \
  ~/  user@backup:/backups/home/
```

#### Example 4: System Configuration Backup
```bash
# Backup /etc with specific includes
rsync -av \
  --include='*/' \
  --include='*.conf' \
  --include='*.cfg' \
  --exclude='*' \
  /etc/ /backup/etc-$(date +%Y%m%d)/
```

#### Example 5: Log File Sync
```bash
# Sync logs, keep last 7 days
rsync -av --delete \
  --exclude="*.gz" \
  --min-age=7 \
  /var/log/ user@log-server:/logs/$(hostname)/
```

#### Example 6: Two-Way Sync (Careful!)
```bash
# Sync local to remote
rsync -av --delete /local/shared/ user@remote:/remote/shared/

# Then sync back (be very careful with --delete!)
rsync -av /remote/shared/ user@remote:/local/shared/
```

### Comparing SCP vs RSYNC:

```bash
# SCP: Simple one-time copy
scp -r /var/www/ user@remote:/backup/

# RSYNC: Smart sync (only copies differences)
rsync -av /var/www/ user@remote:/backup/

# Second rsync is MUCH faster (only syncs changes)
rsync -av /var/www/ user@remote:/backup/
```

### Advanced RSYNC Features:

#### Checksum-Based Sync:
```bash
# Use checksum instead of timestamp/size
rsync -avc /local/dir/ user@remote:/remote/dir/
```

#### Update Only (Don't Overwrite Newer):
```bash
# Only update files that are older
rsync -avu /local/dir/ user@remote:/remote/dir/
```

#### Preserve Hard Links:
```bash
# Preserve hard links
rsync -avH /local/dir/ user@remote:/remote/dir/
```

#### Backup with Versioning:
```bash
# Keep old versions in backup directory
rsync -av --backup --backup-dir=/backup/old-$(date +%Y%m%d) \
  /local/dir/ user@remote:/remote/dir/
```

#### Mirror Entire Directory:
```bash
# Create exact mirror (be careful with --delete)
rsync -av --delete --force /source/ /destination/
```

### Testing and Verification:

```bash
# Create test environment
mkdir -p /tmp/source/{dir1,dir2,dir3}
echo "file1" > /tmp/source/file1.txt
echo "file2" > /tmp/source/dir1/file2.txt
echo "file3" > /tmp/source/dir2/file3.log

# Test scp
scp -r /tmp/source/ user@remote:/tmp/dest-scp/

# Test rsync dry-run
rsync -avn /tmp/source/ user@remote:/tmp/dest-rsync/

# Test rsync actual
rsync -av /tmp/source/ user@remote:/tmp/dest-rsync/

# Verify
ssh user@remote "ls -R /tmp/dest-rsync/"

# Test exclude
rsync -av --exclude='*.log' /tmp/source/ user@remote:/tmp/dest-rsync-no-logs/

# Verify exclude worked
ssh user@remote "find /tmp/dest-rsync-no-logs/ -name '*.log'"
```

### Create Backup Script:
```bash
# Create automated backup script
cat > ~/backup.sh << 'EOF'
#!/bin/bash

SOURCE="/home/data"
DEST="user@backup-server:/backups/$(hostname)"
DATE=$(date +%Y%m%d)
LOG="/var/log/backup-${DATE}.log"

echo "Starting backup at $(date)" | tee -a $LOG

rsync -avz --delete \
  --exclude='*.tmp' \
  --exclude='.cache/' \
  --log-file=$LOG \
  --stats \
  $SOURCE $DEST

if [ $? -eq 0 ]; then
    echo "Backup completed successfully at $(date)" | tee -a $LOG
else
    echo "Backup failed at $(date)" | tee -a $LOG
    exit 1
fi
EOF

chmod +x ~/backup.sh

# Test
~/backup.sh
```

### Verification Checklist:
```bash
# Test scp file transfer
scp /etc/hosts user@remote:/tmp/
ssh user@remote "cat /tmp/hosts"

# Test scp directory transfer
scp -r /tmp/test/ user@remote:/tmp/
ssh user@remote "ls -R /tmp/test/"

# Test rsync sync
rsync -av /tmp/test/ user@remote:/tmp/test-rsync/
ssh user@remote "ls -R /tmp/test-rsync/"

# Test rsync exclude
rsync -av --exclude='*.log' /var/log/ user@remote:/tmp/logs/

# Test rsync delete
# (Create extra file on remote first, then sync with --delete)
```

</details>

---

## ✅ DRILL D COMPLETION CHECKLIST

- [ ] Tar archives created successfully
- [ ] Gzip compression working
- [ ] Bzip2 compression working
- [ ] XZ compression working
- [ ] Archive extraction successful
- [ ] Compression ratios compared
- [ ] Backup script created
- [ ] ACL installed and working (getfacl/setfacl)
- [ ] User-specific ACLs set
- [ ] Group ACLs configured
- [ ] Default ACLs tested
- [ ] ACLs applied recursively
- [ ] Persistent journal storage configured
- [ ] Journalctl filtering mastered
- [ ] Logs exported and analyzed
- [ ] Grep with regex patterns working
- [ ] IP/email patterns extracted
- [ ] Log analysis performed
- [ ] SCP file transfers successful
- [ ] Rsync syncing working
- [ ] Rsync with excludes tested
- [ ] Remote backups automated

**Time Completed: _____ minutes**

---

## 💪 MASTERY INDICATORS

✅ Complete in < 60 minutes
✅ Can create compressed archives efficiently
✅ Understand compression trade-offs (speed vs size)
✅ ACLs persist after reboot
✅ Can filter logs efficiently
✅ Regex patterns match correctly
✅ Remote transfers work seamlessly
✅ Backup procedures automated
✅ All configurations documented