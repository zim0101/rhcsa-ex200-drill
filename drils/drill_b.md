# RHCSA DRILL B - Services & Scripting
**Time Target: 95 minutes | Days 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30**

---

## 🎯 SCENARIO 1: PERFORMANCE TUNING (10 minutes)

### THE SITUATION:
Optimize server performance using tuned profiles based on workload requirements.

### YOUR TASKS:
1. Check current tuned profile
2. List available profiles
3. Switch to throughput-performance profile
4. Get tuned recommendation
5. Verify profile is active

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Check Tuned Status:
```bash
sudo systemctl enable --now tuned
tuned-adm active
tuned-adm recommend
```

### List and Switch Profiles:
```bash
# List all profiles
tuned-adm list

# Switch profile
sudo tuned-adm profile throughput-performance

# Verify
tuned-adm active
```

### Verification:
```bash
tuned-adm active
tuned-adm verify
cat /etc/tuned/active_profile
```
</details>

---

## 🎯 SCENARIO 2: TIME SYNCHRONIZATION WITH CHRONY (10 minutes)

### THE SITUATION:
Configure accurate time synchronization for the server using chrony and set the correct timezone.

### YOUR TASKS:
1. Configure chrony with NTP servers: `0.rhel.pool.ntp.org`, `1.rhel.pool.ntp.org`
2. Set timezone to `America/New_York`
3. Enable NTP synchronization
4. Verify time sync is working

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Configure Chrony:
```bash
# Backup config
sudo cp /etc/chrony.conf /etc/chrony.conf.backup

# Edit configuration
sudo vi /etc/chrony.conf
```

Add/Modify:
```conf
server 0.rhel.pool.ntp.org iburst
server 1.rhel.pool.ntp.org iburst
server 2.rhel.pool.ntp.org iburst

makestep 1.0 3
rtcsync
```

```bash
# Restart chronyd
sudo systemctl restart chronyd
sudo systemctl status chronyd
```

### Set Timezone:
```bash
# List timezones
timedatectl list-timezones | grep America

# Set timezone
sudo timedatectl set-timezone America/New_York

# Verify
timedatectl
```

### Enable NTP:
```bash
sudo timedatectl set-ntp true
timedatectl status
```

### Verify Sync:
```bash
chronyc sources -v
chronyc tracking
timedatectl timesync-status
```

### Verification:
```bash
timedatectl status
chronyc sources
systemctl status chronyd
```
</details>

---

## 🎯 SCENARIO 3: SERVICE MANAGEMENT (20 minutes)

### THE SITUATION:
Manage system services: configure Apache, create custom monitoring service, and mask unused services.

### YOUR TASKS:
1. Install and configure Apache (httpd)
2. Ensure httpd starts automatically on boot
3. Create custom systemd service at `/opt/monitor/app.sh`
4. Mask the `atd` service
5. Verify all configurations persist

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Install and Configure Apache:
```bash
sudo yum install httpd -y
sudo systemctl enable --now httpd

# Verify
sudo systemctl status httpd
sudo systemctl is-enabled httpd
sudo systemctl is-active httpd
```

### Create Custom Service:
```bash
# Create application
sudo mkdir -p /opt/monitor
sudo vi /opt/monitor/app.sh
```

Add:
```bash
#!/bin/bash
while true; do
    echo "$(date): Monitoring app running" >> /var/log/monitor.log
    sleep 60
done
```

```bash
sudo chmod +x /opt/monitor/app.sh

# Create service file
sudo vi /etc/systemd/system/monitor.service
```

Add:
```ini
[Unit]
Description=Custom Monitoring Application
After=network.target

[Service]
Type=simple
ExecStart=/opt/monitor/app.sh
Restart=on-failure
RestartSec=10s
User=root

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now monitor.service
sudo systemctl status monitor.service
```

### Mask atd Service:
```bash
sudo systemctl mask atd
sudo systemctl status atd
```

### Verification:
```bash
sudo systemctl status httpd
sudo systemctl status monitor
tail /var/log/monitor.log
sudo systemctl is-enabled atd    # Should show: masked
```
</details>

---

## 🎯 SCENARIO 4: JOB SCHEDULING (15 minutes)

### THE SITUATION:
Automate system tasks: database backups at 2 AM daily, log cleanup on Sundays, health checks every 15 minutes, and a one-time security scan.

### YOUR TASKS:
1. Create cron job for daily 2 AM backup
2. Schedule log cleanup for Sundays at midnight
3. Run health check every 15 minutes
4. Schedule one-time job tomorrow at 3 PM using `at`
5. Verify all scheduled jobs

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Create Backup Script:
```bash
sudo vi /usr/local/bin/backup.sh
```

Add:
```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
tar -czf /backup/db-backup-$DATE.tar.gz /var/lib/mysql/
echo "$(date): Backup completed" >> /var/log/backup.log
```

```bash
sudo chmod +x /usr/local/bin/backup.sh
sudo mkdir -p /backup
```

### Configure Cron Jobs:
```bash
crontab -e
```

Add:
```
# Daily backup at 2:00 AM
0 2 * * * /usr/local/bin/backup.sh

# Log cleanup every Sunday at midnight
0 0 * * 0 /usr/local/bin/cleanup-logs.sh

# Health check every 15 minutes
*/15 * * * * /usr/local/bin/health-check.sh
```

### Create Additional Scripts:
```bash
# Cleanup script
sudo vi /usr/local/bin/cleanup-logs.sh
```

Add:
```bash
#!/bin/bash
find /var/log -name "*.log" -mtime +30 -exec rm {} \;
echo "$(date): Logs cleaned" >> /var/log/cleanup.log
```

```bash
# Health check script
sudo vi /usr/local/bin/health-check.sh
```

Add:
```bash
#!/bin/bash
CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}')
MEM=$(free | grep Mem | awk '{print ($3/$2)*100}')
echo "$(date): CPU=${CPU}% MEM=${MEM}%" >> /var/log/health.log
```

```bash
sudo chmod +x /usr/local/bin/cleanup-logs.sh
sudo chmod +x /usr/local/bin/health-check.sh
```

### Schedule One-Time Job:
```bash
sudo yum install at -y
sudo systemctl enable --now atd

# Schedule job
at 15:00 tomorrow
```

Type and press Ctrl+D:
```
/usr/local/bin/security-scan.sh
```

```bash
# List jobs
atq

# View job details
at -c <job-number>
```

### Verification:
```bash
crontab -l
sudo systemctl status crond
atq
grep CRON /var/log/cron
```
</details>

---

## 🎯 SCENARIO 5: PROCESS MANAGEMENT (15 minutes)

### THE SITUATION:
Identify resource-intensive processes, adjust priorities, and manage process lifecycle.

### YOUR TASKS:
1. Identify top 5 CPU and memory-consuming processes
2. Find and kill a process by name
3. Start a process with low priority (nice 10)
4. Change priority of running process
5. Use job control (background/foreground)

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### View Processes:
```bash
# Snapshot view
ps aux
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10

# Interactive monitoring
top
# Inside top: M (memory), P (CPU), k (kill), r (renice), q (quit)
```

### Find and Kill Processes:
```bash
# Find by name
pgrep firefox
ps aux | grep firefox

# Kill by PID
kill 1234              # SIGTERM
kill -9 1234           # SIGKILL

# Kill by name
killall firefox
pkill firefox
```

### Process Priority:
```bash
# Start with low priority
nice -n 10 /usr/local/bin/batch-job.sh &

# View nice values
ps -eo pid,ni,cmd | grep batch-job

# Change priority
PID=$(pgrep batch-job)
renice 15 $PID

# High priority (requires sudo)
sudo renice -5 $PID
```

### Job Control:
```bash
# Start in background
sleep 300 &

# List jobs
jobs
jobs -l

# Suspend foreground job (Ctrl+Z)
sleep 300
# Press Ctrl+Z

# Resume in background
bg %1

# Bring to foreground
fg %1
```

### Verification:
```bash
top
ps aux --sort=-%cpu | head -10
jobs
```
</details>

---

## 🎯 SCENARIO 6: BASH SCRIPTING WITH ARGUMENTS & INPUT (25 minutes)

### THE SITUATION:
Create production-ready scripts with proper argument handling, validation, and error checking.

### YOUR TASKS:
1. Create backup script accepting source and destination arguments
2. Create user report script with UID threshold argument
3. Create interactive menu script
4. Implement proper error handling and exit codes
5. Test all scripts thoroughly

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Script 1: Backup with Arguments
```bash
vi ~/backup-with-args.sh
```

Add:
```bash
#!/bin/bash

# Check arguments
if [ $# -lt 2 ]; then
    echo "Usage: $0 <source> <destination>"
    echo "Example: $0 /home/user/documents /backup"
    exit 1
fi

SOURCE=$1
DEST=$2
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Validate source
if [ ! -d "$SOURCE" ]; then
    echo "Error: Source '$SOURCE' does not exist"
    exit 2
fi

# Create destination
mkdir -p "$DEST" || {
    echo "Error: Cannot create destination"
    exit 3
}

# Perform backup
BACKUP_FILE="${DEST}/backup_${TIMESTAMP}.tar.gz"
echo "Backing up $SOURCE..."

if tar -czf "$BACKUP_FILE" "$SOURCE" 2>/dev/null; then
    echo "✓ Backup successful: $BACKUP_FILE"
    echo "  Size: $(du -h "$BACKUP_FILE" | cut -f1)"
    exit 0
else
    echo "✗ Backup failed"
    exit 4
fi
```

```bash
chmod +x ~/backup-with-args.sh

# Test
./backup-with-args.sh /etc /tmp/test
echo "Exit code: $?"
```

### Script 2: User Report with Arguments
```bash
vi ~/user-report-args.sh
```

Add:
```bash
#!/bin/bash

UID_THRESHOLD=1000

# Process arguments
while [ $# -gt 0 ]; do
    case $1 in
        -u|--uid)
            UID_THRESHOLD="$2"
            shift 2
            ;;
        -h|--help)
            echo "Usage: $0 [OPTIONS]"
            echo "  -u, --uid <number>    UID threshold (default: 1000)"
            echo "  -h, --help            Show help"
            exit 0
            ;;
        *)
            echo "Unknown option: $1"
            exit 1
            ;;
    esac
done

# Validate UID is number
if ! [[ "$UID_THRESHOLD" =~ ^[0-9]+$ ]]; then
    echo "Error: UID must be a number"
    exit 1
fi

echo "======================================="
echo "User Report - $(date)"
echo "UID Threshold: >= $UID_THRESHOLD"
echo "======================================="

count=0

while IFS=: read -r username password uid gid comment home shell; do
    if [ "$uid" -ge "$UID_THRESHOLD" ] && [ "$uid" != 65534 ]; then
        echo "Username: $username"
        echo "  UID: $uid"
        echo "  Home: $home"
        echo "  Shell: $shell"
        echo ""
        count=$((count + 1))
    fi
done < /etc/passwd

echo "Total users: $count"
exit 0
```

```bash
chmod +x ~/user-report-args.sh

# Test
./user-report-args.sh
./user-report-args.sh --uid 500
./user-report-args.sh -h
```

### Script 3: Interactive Menu
```bash
vi ~/interactive-menu.sh
```

Add:
```bash
#!/bin/bash

while true; do
    clear
    echo "================================"
    echo "   System Administration Menu"
    echo "================================"
    echo "1. View disk usage"
    echo "2. View memory usage"
    echo "3. View logged in users"
    echo "4. Backup directory"
    echo "5. Create user"
    echo "6. Exit"
    echo "================================"
    
    read -p "Enter choice [1-6]: " choice
    
    case $choice in
        1)
            echo ""
            df -h
            read -p "Press Enter to continue..."
            ;;
        2)
            echo ""
            free -h
            read -p "Press Enter to continue..."
            ;;
        3)
            echo ""
            who
            w
            read -p "Press Enter to continue..."
            ;;
        4)
            echo ""
            read -p "Enter source directory: " source
            read -p "Enter destination: " dest
            
            if [ -d "$source" ]; then
                timestamp=$(date +%Y%m%d_%H%M%S)
                tar -czf "${dest}/backup_${timestamp}.tar.gz" "$source"
                echo "Backup completed!"
            else
                echo "Error: Source not found"
            fi
            read -p "Press Enter to continue..."
            ;;
        5)
            echo ""
            read -p "Enter username: " username
            read -sp "Enter password: " password
            echo ""
            
            if sudo useradd "$username"; then
                echo "$password" | sudo passwd --stdin "$username"
                echo "User created successfully"
            else
                echo "Error creating user"
            fi
            read -p "Press Enter to continue..."
            ;;
        6)
            echo "Goodbye!"
            exit 0
            ;;
        *)
            echo "Invalid option"
            sleep 2
            ;;
    esac
done
```

```bash
chmod +x ~/interactive-menu.sh
./interactive-menu.sh
```

### Script 4: Command Output Processing
```bash
vi ~/process-output.sh
```

Add:
```bash
#!/bin/bash

# Capture command output
HOSTNAME=$(hostname)
KERNEL=$(uname -r)
UPTIME=$(uptime -p)
CPU_COUNT=$(nproc)
MEMORY_TOTAL=$(free -h | awk '/^Mem:/{print $2}')

echo "System Information Report"
echo "========================="
echo "Hostname: $HOSTNAME"
echo "Kernel: $KERNEL"
echo "Uptime: $UPTIME"
echo "CPU Cores: $CPU_COUNT"
echo "Total Memory: $MEMORY_TOTAL"
echo ""

# Process file line by line
echo "System users (UID >= 1000):"
while IFS=: read -r user x uid rest; do
    if [ "$uid" -ge 1000 ] && [ "$uid" != 65534 ]; then
        echo "  - $user (UID: $uid)"
    fi
done < /etc/passwd

# Process command output
echo ""
echo "Top 5 largest directories in /var:"
du -sh /var/* 2>/dev/null | sort -rh | head -5

exit 0
```

```bash
chmod +x ~/process-output.sh
./process-output.sh
```

### Verification:
```bash
# Test all scripts
./backup-with-args.sh /etc /tmp/test
./user-report-args.sh --uid 1000
./process-output.sh
ls -l ~/*.sh
```
</details>

---

## ✅ DRILL B COMPLETION CHECKLIST

- [ ] Tuned profile changed to throughput-performance
- [ ] Chrony configured and time synced
- [ ] Timezone set correctly
- [ ] Apache service running and enabled
- [ ] Custom monitor service created
- [ ] atd service masked
- [ ] Cron jobs scheduled
- [ ] at job scheduled
- [ ] Process management practiced
- [ ] Scripts with arguments created
- [ ] Interactive menu working
- [ ] All scripts tested successfully
- [ ] System rebooted - services persist

**Time Completed: _____ minutes**

---

## 💪 MASTERY INDICATORS

✅ Complete in < 95 minutes
✅ All services start on boot
✅ Scripts handle arguments correctly
✅ Proper error handling implemented
✅ Exit codes used appropriately
✅ Cron jobs execute as scheduled