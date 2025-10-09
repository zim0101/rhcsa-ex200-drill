# RHCSA DRILL B - Services, Automation & Scripting
**Time Target: 90-120 minutes | Days 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30**

---

## 🎯 SCENARIO 1: SERVICE MANAGEMENT (20 minutes)

### THE SITUATION:
You're managing a web server that needs Apache HTTP server (httpd) running at all times. The development team also created a custom monitoring application that should run as a system service. Previous admin left the `atd` service enabled but it's not needed for production.

### YOUR TASKS:
1. Install and configure Apache (httpd)
2. Ensure httpd starts automatically on boot and start it now
3. Create a custom systemd service for the monitoring app at `/opt/monitor/app.sh`
4. Prevent the `atd` service from ever starting (mask it)
5. Verify all service configurations persist after reboot

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Install and Configure Apache:
```bash
# Install Apache
sudo yum install httpd -y

# Start service immediately
sudo systemctl start httpd

# Enable to start at boot
sudo systemctl enable httpd

# Or combine both:
sudo systemctl enable --now httpd

# Check status
sudo systemctl status httpd
sudo systemctl is-enabled httpd                # Should show: enabled
sudo systemctl is-active httpd                 # Should show: active
```

### Create Custom Application Service:
```bash
# Create application directory
sudo mkdir -p /opt/monitor

# Create the application script
sudo vi /opt/monitor/app.sh
```

Add this content:
```bash
#!/bin/bash
while true; do
    echo "$(date): Monitoring app running" >> /var/log/monitor.log
    sleep 60
done
```

```bash
# Make executable
sudo chmod +x /opt/monitor/app.sh

# Create systemd service file
sudo vi /etc/systemd/system/monitor.service
```

Add this content:
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
# Reload systemd to recognize new service
sudo systemctl daemon-reload

# Enable and start the service
sudo systemctl enable --now monitor.service

# Check status
sudo systemctl status monitor.service

# View logs
sudo tail -f /var/log/monitor.log              # Ctrl+C to exit
```

### Mask the atd Service:
```bash
# Check if atd is running
sudo systemctl status atd

# Mask it (prevents starting)
sudo systemctl mask atd

# Try to start it (should fail)
sudo systemctl start atd
# Output: Failed to start atd.service: Unit atd.service is masked.

# Verify it's masked
sudo systemctl is-enabled atd                  # Should show: masked
```

### Additional Service Management Commands:
```bash
# Stop a service
sudo systemctl stop httpd

# Restart a service
sudo systemctl restart httpd

# Reload configuration without restart
sudo systemctl reload httpd

# View all services
sudo systemctl list-units --type=service
sudo systemctl list-unit-files --type=service

# View failed services
sudo systemctl --failed

# If you need to unmask:
# sudo systemctl unmask atd
```

### Verification After Reboot:
```bash
sudo systemctl reboot

# After reboot:
sudo systemctl status httpd                    # Should be active
sudo systemctl status monitor                  # Should be active
sudo systemctl status atd                      # Should be masked
tail /var/log/monitor.log                      # Should show continuous logs
```
</details>

---

## 🎯 SCENARIO 2: JOB SCHEDULING (15 minutes)

### THE SITUATION:
Your organization needs several automated tasks: database backups at 2 AM daily, log cleanup every Sunday at midnight, system health checks every 15 minutes, and a one-time security scan tomorrow at 3 PM.

### YOUR TASKS:
1. Create a cron job to run backup script daily at 2:00 AM
2. Schedule log cleanup for Sundays at midnight
3. Run health check every 15 minutes
4. Schedule one-time security scan for tomorrow 3 PM using `at`
5. Verify all scheduled jobs

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Create Backup Script:
```bash
# Create backup script
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
# Edit user crontab
crontab -e

# If prompted, choose editor (vim recommended)
```

Add these lines:
```
# Daily backup at 2:00 AM
0 2 * * * /usr/local/bin/backup.sh

# Log cleanup every Sunday at midnight
0 0 * * 0 /usr/local/bin/cleanup-logs.sh

# Health check every 15 minutes
*/15 * * * * /usr/local/bin/health-check.sh

# Alternative: Run on weekdays at 8:30 AM
# 30 8 * * 1-5 /usr/local/bin/morning-report.sh

# Run on first day of month
# 0 0 1 * * /usr/local/bin/monthly-report.sh
```

```bash
# View crontab
crontab -l

# Cron syntax reminder:
# * * * * *  command
# │ │ │ │ │
# │ │ │ │ └─── Day of week (0-7, 0 and 7 are Sunday)
# │ │ │ └───── Month (1-12)
# │ │ └─────── Day of month (1-31)
# │ └───────── Hour (0-23)
# └─────────── Minute (0-59)
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

### System-Wide Cron (as root):
```bash
# Edit system crontab
sudo vi /etc/crontab

# Or place scripts in:
sudo ls /etc/cron.{hourly,daily,weekly,monthly}/
sudo cp /usr/local/bin/backup.sh /etc/cron.daily/
```

### Schedule One-Time Job with at:
```bash
# Install at if needed
sudo yum install at -y
sudo systemctl enable --now atd

# Schedule job for tomorrow 3 PM
at 15:00 tomorrow
```

Type commands, then press Ctrl+D:
```
/usr/local/bin/security-scan.sh
```

Alternative scheduling examples:
```bash
# Various at scheduling options:
at now + 2 minutes
at 23:30
at 10:00 AM tomorrow
at 2:00 PM 12/25/2025
at now + 1 hour
at noon
at midnight
```

```bash
# List scheduled at jobs
atq

# View specific job details
at -c <job-number>

# Remove scheduled job
atrm <job-number>
```

### Anacron for Systems Not Always On:
```bash
# View anacron configuration
cat /etc/anacrontab

# Anacron ensures jobs run even if system was off
# Format: period delay job-identifier command
# Example: 7 10 weekly_backup /usr/local/bin/backup.sh
```

### Verification:
```bash
# View crontab
crontab -l

# Check cron service
sudo systemctl status crond

# View cron logs
sudo grep CRON /var/log/cron

# View at queue
atq

# Test script manually
/usr/local/bin/backup.sh
cat /var/log/backup.log
```
</details>

---

## 🎯 SCENARIO 3: PROCESS MANAGEMENT (15 minutes)

### THE SITUATION:
Your server is experiencing performance issues. You need to identify resource-intensive processes, adjust priorities, and kill problematic processes. A runaway process is consuming 100% CPU, and you need to nice a batch job that's running too aggressively.

### YOUR TASKS:
1. Identify the top 5 CPU-consuming processes
2. Identify the top 5 memory-consuming processes
3. Find a specific process by name and kill it
4. Start a process with low priority (nice value 10)
5. Change the priority of a running process
6. Use job control (background/foreground)

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### View Processes:
```bash
# Snapshot view
ps aux                                         # All processes
ps aux | head -20                              # Top 20 lines
ps -ef                                         # Full format

# View specific user's processes
ps -u john
ps aux | grep httpd

# View with custom columns
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -10
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -10

# Tree view
ps auxf                                        # ASCII tree
pstree                                         # Process tree
pstree -p                                      # With PIDs
```

### Interactive Process Monitoring:
```bash
# Launch top
top

# Inside top (press keys):
# M - Sort by memory
# P - Sort by CPU
# k - Kill process (enter PID)
# r - Renice process
# q - Quit

# Alternative: htop (install if available)
# sudo yum install htop -y
# htop                                         # More user-friendly
```

### Find and Kill Processes:
```bash
# Find process by name
pgrep firefox
pgrep -l firefox                               # With process name
ps aux | grep firefox

# Kill by PID
kill 1234                                      # SIGTERM (graceful)
kill -15 1234                                  # SIGTERM (explicit)
kill -9 1234                                   # SIGKILL (force)

# Kill by name
killall firefox                                # Kill all instances
pkill firefox                                  # Pattern kill

# Kill all processes by user
pkill -u john
```

### List All Kill Signals:
```bash
kill -l

# Common signals:
# 1  SIGHUP   - Hangup (reload config)
# 2  SIGINT   - Interrupt (Ctrl+C)
# 9  SIGKILL  - Kill (cannot be caught)
# 15 SIGTERM  - Terminate (graceful shutdown)
# 18 SIGCONT  - Continue if stopped
# 19 SIGSTOP  - Stop (cannot be caught)
```

### Process Priority (Nice Values):
```bash
# Nice values: -20 (highest priority) to 19 (lowest priority)
# Default is 0

# Start process with low priority
nice -n 10 /usr/local/bin/batch-job.sh &

# Start with high priority (requires sudo)
sudo nice -n -10 /usr/local/bin/critical-job.sh &

# View nice values
ps -eo pid,ni,cmd | grep batch-job

# Change priority of running process
PID=$(pgrep batch-job)
echo "PID: $PID"

# Renice (increase nice value = lower priority)
renice 15 $PID

# High priority (requires sudo)
sudo renice -5 $PID

# Verify
ps -eo pid,ni,cmd | grep batch-job
```

### Job Control (Background/Foreground):
```bash
# Start job in background
sleep 300 &
[1] 12345                                      # Job number and PID

# List background jobs
jobs
jobs -l                                        # With PID

# Start in foreground, then suspend
sleep 300
# Press Ctrl+Z to suspend
[1]+ Stopped    sleep 300

# Resume in background
bg %1                                          # Job 1
bg                                             # Most recent job

# Bring to foreground
fg %1
fg                                             # Most recent job

# Example workflow:
vim bigfile.txt
# Press Ctrl+Z
# [1]+ Stopped    vim bigfile.txt
jobs
# Continue editing later:
fg %1
```

### Find Open Files:
```bash
# Install lsof if needed
sudo yum install lsof -y

# All open files
lsof | head -20

# Files opened by specific user
lsof -u john

# Files opened by specific process
lsof -c httpd

# Network files (ports)
lsof -i                                        # All network files
lsof -i :80                                    # Port 80
lsof -i tcp                                    # TCP connections

# Files in specific directory
lsof +D /var/log

# Find which process is using a file
lsof /var/log/messages
```

### Advanced Process Investigation:
```bash
# View process details
cat /proc/<PID>/status
cat /proc/<PID>/cmdline
cat /proc/<PID>/environ

# View process tree
ps -ejH                                        # Hierarchy
ps axjf                                        # Tree format

# View threads
ps -eLf                                        # All threads
top -H                                         # Thread view in top
```

### Verification:
```bash
# Monitor system
top
htop                                           # If installed
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10
lsof | wc -l                                   # Count open files
```
</details>

---

## 🎯 SCENARIO 4: PERFORMANCE TUNING (10 minutes)

### THE SITUATION:
Your server needs performance optimization. You need to apply appropriate tuned profiles based on workload and create a custom profile for specific requirements.

### YOUR TASKS:
1. Check current tuned profile
2. List available profiles
3. Switch to throughput-performance profile
4. Get tuned recommendation for your system
5. Create a custom profile based on balanced

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Check Tuned Status:
```bash
# Ensure tuned is running
sudo systemctl status tuned
sudo systemctl enable --now tuned

# Check active profile
tuned-adm active

# Get system recommendation
tuned-adm recommend
```

### List Available Profiles:
```bash
# List all profiles
tuned-adm list

# Common profiles:
# - balanced: Default, balanced performance/power
# - throughput-performance: Maximum throughput
# - latency-performance: Low latency
# - network-latency: Optimized for network latency
# - network-throughput: Optimized for network throughput
# - powersave: Power saving mode
# - virtual-guest: For virtual machines
# - virtual-host: For hypervisors
```

### Switch Profile:
```bash
# Switch to throughput-performance
sudo tuned-adm profile throughput-performance

# Verify
tuned-adm active

# Switch back to balanced
sudo tuned-adm profile balanced

# Disable tuned (not recommended)
# sudo tuned-adm off
```

### Create Custom Profile:
```bash
# Create profile directory
sudo mkdir -p /etc/tuned/custom-profile

# Create configuration file
sudo vi /etc/tuned/custom-profile/tuned.conf
```

Add:
```ini
[main]
summary=Custom profile based on balanced
include=balanced

[vm]
transparent_hugepages=never

[sysctl]
net.ipv4.tcp_fastopen=3
vm.swappiness=10

[disk]
elevator=deadline
```

```bash
# Activate custom profile
sudo tuned-adm profile custom-profile

# Verify
tuned-adm active
```

### View Profile Details:
```bash
# View profile information
tuned-adm profile_info balanced
tuned-adm profile_info throughput-performance
```

### Verification:
```bash
tuned-adm active
tuned-adm list
tuned-adm verify                               # Verify settings applied
cat /etc/tuned/active_profile
```
</details>

---

## 🎯 SCENARIO 5: BOOTLOADER CONFIGURATION (10 minutes)

### THE SITUATION:
You need to modify the GRUB bootloader to change the timeout, add kernel parameters, and set a specific boot entry as default.

### YOUR TASKS:
1. Increase GRUB timeout to 10 seconds
2. Remove "rhgb quiet" to see boot messages
3. Add "net.ifnames=0" kernel parameter
4. Regenerate GRUB configuration
5. Set first entry as default

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Backup GRUB Configuration:
```bash
sudo cp /etc/default/grub /etc/default/grub.backup
```

### Edit GRUB Configuration:
```bash
# Edit GRUB defaults
sudo vi /etc/default/grub
```

Current content (example):
```
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto resume=UUID=xxx rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

Modify to:
```
GRUB_TIMEOUT=10
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=0
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto resume=UUID=xxx rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap net.ifnames=0"
GRUB_DISABLE_RECOVERY="true"
```

Key changes:
- `GRUB_TIMEOUT=10` - 10 second timeout
- `GRUB_DEFAULT=0` - First entry is default
- Removed `rhgb quiet` - See boot messages
- Added `net.ifnames=0` - Traditional network naming

### Regenerate GRUB Configuration:
```bash
# For BIOS systems:
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# For EFI systems:
sudo grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
# Or for RHEL:
# sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# View generated config (optional)
sudo cat /boot/grub2/grub.cfg | less
```

### Set Default Boot Entry:
```bash
# List all boot entries
sudo grub2-editenv list

# Set default to first entry
sudo grub2-set-default 0

# Verify
sudo grub2-editenv list
```

### Alternative: Set Default by Name:
```bash
# List menu entries with numbers
sudo awk -F\' '/menuentry / {print $2}' /boot/grub2/grub.cfg

# Set by index
sudo grub2-set-default 0
```

### Common Kernel Parameters:
```bash
# Examples of useful kernel parameters:
# net.ifnames=0 biosdevname=0  - Traditional network names (eth0)
# console=ttyS0                - Serial console
# rd.break                     - Break into emergency shell
# init=/bin/bash               - Boot to bash shell (RHEL 9)
# selinux=0                    - Disable SELinux at boot
# systemd.unit=rescue.target   - Boot to rescue mode
# systemd.unit=emergency.target - Boot to emergency mode
```

### Verification After Reboot:
```bash
sudo systemctl reboot

# After reboot:
cat /proc/cmdline                              # View actual kernel parameters
dmesg | head -50                               # Should see boot messages now
```

### Troubleshooting GRUB:
```bash
# If system won't boot, use rescue mode:
# 1. Boot from installation media
# 2. Choose "Troubleshooting" → "Rescue"
# 3. chroot /mnt/sysroot
# 4. grub2-mkconfig -o /boot/grub2/grub.cfg
# 5. grub2-install /dev/sda (for BIOS)
```
</details>

---

## 🎯 SCENARIO 6: BASH SCRIPTING (25 minutes)

### THE SITUATION:
You need to create automation scripts for: system backup, user report generation, and service monitoring. Scripts must handle arguments, conditionals, loops, and error checking.

### YOUR TASKS:
1. Create a backup script that accepts a directory argument
2. Create a user report script showing users with UID ≥ 1000
3. Create a service monitor script that checks multiple services
4. Use if/else, for loops, and proper error handling
5. Test all scripts with various inputs

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Script 1: Backup Script with Arguments
```bash
vi ~/backup-dir.sh
```

Add:
```bash
#!/bin/bash

# Check if argument provided
if [ $# -eq 0 ]; then
    echo "Usage: $0 <directory>"
    echo "Example: $0 /home/user/documents"
    exit 1
fi

# Get directory from argument
DIR=$1

# Check if directory exists
if [ ! -d "$DIR" ]; then
    echo "Error: Directory '$DIR' does not exist"
    exit 1
fi

# Create backup filename with timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="/backup/backup_${TIMESTAMP}.tar.gz"

# Create backup directory if it doesn't exist
sudo mkdir -p /backup

# Perform backup
echo "Backing up $DIR..."
if tar -czf "$BACKUP_FILE" "$DIR" 2>/dev/null; then
    echo "Backup successful: $BACKUP_FILE"
    ls -lh "$BACKUP_FILE"
else
    echo "Error: Backup failed"
    exit 1
fi

# Cleanup old backups (keep last 5)
cd /backup
ls -t backup_*.tar.gz | tail -n +6 | xargs -r rm -f
echo "Old backups cleaned up"
```

```bash
chmod +x ~/backup-dir.sh

# Test
./backup-dir.sh /etc
./backup-dir.sh /nonexistent                   # Should show error
```

### Script 2: User Report
```bash
vi ~/user-report.sh
```

Add:
```bash
#!/bin/bash

# User report for UID >= 1000
echo "==================================="
echo "User Report - $(date)"
echo "==================================="
echo ""

# Counter
count=0

# Read /etc/passwd
while IFS=: read -r username password uid gid comment home shell; do
    # Check if UID >= 1000 and not nobody
    if [ "$uid" -ge 1000 ] && [ "$uid" != 65534 ]; then
        echo "Username: $username"
        echo "  UID: $uid"
        echo "  Home: $home"
        echo "  Shell: $shell"
        
        # Check if user is logged in
        if who | grep -q "^$username "; then
            echo "  Status: LOGGED IN"
        else
            echo "  Status: Not logged in"
        fi
        
        # Check last login
        lastlog -u "$username" 2>/dev/null | tail -1
        
        echo ""
        count=$((count + 1))
    fi
done < /etc/passwd

echo "==================================="
echo "Total users: $count"
echo "==================================="
```

```bash
chmod +x ~/user-report.sh
./user-report.sh
```

### Script 3: Service Monitor
```bash
vi ~/service-monitor.sh
```

Add:
```bash
#!/bin/bash

# List of services to monitor
SERVICES=("sshd" "httpd" "crond" "firewalld")

echo "Service Monitoring Report - $(date)"
echo "========================================"

# Track statistics
total=0
running=0
stopped=0

# Loop through services
for service in "${SERVICES[@]}"; do
    total=$((total + 1))
    
    echo -n "Checking $service... "
    
    if systemctl is-active --quiet "$service"; then
        echo "✓ RUNNING"
        running=$((running + 1))
    else
        echo "✗ STOPPED"
        stopped=$((stopped + 1))
        
        # Log to file
        echo "$(date): $service is stopped" >> /var/log/service-monitor.log
        
        # Optionally restart (uncomment if desired)
        # echo "  Attempting to restart $service..."
        # sudo systemctl start "$service"
    fi
done

echo "========================================"
echo "Summary:"
echo "  Total services checked: $total"
echo "  Running: $running"
echo "  Stopped: $stopped"
echo "========================================"

# Exit with error if any service is stopped
if [ $stopped -gt 0 ]; then
    exit 1
else
    exit 0
fi
```

```bash
chmod +x ~/service-monitor.sh

# Test
./service-monitor.sh

# Check exit code
echo $?                                        # 0 if all running, 1 if any stopped
```

### Script 4: Advanced - Interactive Menu
```bash
vi ~/system-menu.sh
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
    echo "4. View running processes"
    echo "5. Backup /etc directory"
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
            read -p "Press Enter to continue..."
            ;;
        4)
            echo ""
            ps aux | head -20
            read -p "Press Enter to continue..."
            ;;
        5)
            echo ""
            ./backup-dir.sh /etc
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
chmod +x ~/system-menu.sh
./system-menu.sh
```

### Script 5: Input/Output Redirection Examples
```bash
vi ~/redirection-examples.sh
```

Add:
```bash
#!/bin/bash

# Standard output redirection
echo "This goes to file" > output.txt
echo "This appends" >> output.txt

# Error redirection
ls /nonexistent 2> errors.txt

# Redirect both stdout and stderr
ls /etc /nonexistent > output.txt 2> errors.txt

# Redirect both to same file
ls /etc /nonexistent &> combined.txt
ls /etc /nonexistent > combined.txt 2>&1    # Alternative syntax

# Discard output
ls /etc > /dev/null 2>&1

# Pipe examples
cat /etc/passwd | grep root
ps aux | grep httpd | awk '{print $2}'
cat /var/log/messages | grep error | wc -l

# Here document
cat << EOF > config.txt
Server: localhost
Port: 8080
User: admin
EOF

# Command substitution
HOSTNAME=$(hostname)
DATE=$(date +%Y-%m-%d)
echo "Backup from $HOSTNAME on $DATE" > backup-info.txt
```

### Testing All Scripts:
```bash
# Test with various inputs
./backup-dir.sh /etc
./backup-dir.sh /home
./backup-dir.sh /nonexistent

# Generate reports
./user-report.sh > user-report.txt
cat user-report.txt

# Monitor services
./service-monitor.sh
sudo systemctl stop httpd
./service-monitor.sh                           # Should show httpd stopped
sudo systemctl start httpd
```

### Verification:
```bash
ls -l ~/*.sh
./backup-dir.sh /tmp
./user-report.sh | head -20
./service-monitor.sh
```
</details>

---

## 🎯 SCENARIO 7: TEXT PROCESSING MASTERY (15 minutes)

### THE SITUATION:
You need to analyze log files, extract specific data, and generate reports. Use grep, sed, awk, cut, and sort to process text efficiently.

### YOUR TASKS:
1. Find all error messages in system logs
2. Extract usernames from /etc/passwd
3. Find and replace text in configuration files
4. Count occurrence of specific patterns
5. Sort and remove duplicates from lists

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### GREP - Search and Filter:
```bash
# Basic search
grep "error" /var/log/messages
grep "failed" /var/log/secure

# Case insensitive
grep -i "error" /var/log/messages

# Invert match (exclude lines)
grep -v "^#" /etc/ssh/sshd_config             # Exclude comments
grep -v "^$" /etc/ssh/sshd_config             # Exclude empty lines

# Combined (no comments or empty lines)
grep -v "^#" /etc/ssh/sshd_config | grep -v "^$"

# Show line numbers
grep -n "root" /etc/passwd

# Count matches
grep -c "bash" /etc/passwd

# Recursive search
grep -r "TODO" /home/user/projects
grep -r "password" /etc/ 2>/dev/null

# Match whole word
grep -w "root" /etc/passwd

# Extended regex
grep -E "root|admin" /etc/passwd
grep -E "error|warning|critical" /var/log/messages

# Show context
grep -A 3 "error" /var/log/messages            # 3 lines after
grep -B 3 "error" /var/log/messages            # 3 lines before
grep -C 3 "error" /var/log/messages            # 3 lines before and after

# Multiple patterns
grep -e "root" -e "admin" /etc/passwd
```

### SED - Stream Editor:
```bash
# Substitute (first occurrence)
sed 's/old/new/' file.txt

# Substitute all occurrences
sed 's/old/new/g' file.txt

# In-place edit
sed -i 's/old/new/g' file.txt

# Backup before editing
sed -i.backup 's/old/new/g' file.txt

# Delete lines
sed '/^#/d' file.txt                           # Delete comments
sed '/^$/d' file.txt                           # Delete empty lines
sed '1,5d' file.txt                            # Delete lines 1-5

# Print specific lines
sed -n '10p' file.txt                          # Print line 10
sed -n '10,20p' file.txt                       # Print lines 10-20
sed -n '/pattern/p' file.txt                   # Print matching lines

# Multiple operations
sed -e 's/old1/new1/g' -e 's/old2/new2/g' file.txt

# Example: Clean config file
sed -e '/^#/d' -e '/^$/d' -e 's/  */ /g' /etc/ssh/sshd_config
```

### AWK - Pattern Scanning and Processing:
```bash
# Print columns
awk '{print $1}' /etc/passwd                   # First column
awk '{print $1, $3}' /etc/passwd               # Columns 1 and 3

# Custom delimiter
awk -F: '{print $1}' /etc/passwd               # Username
awk -F: '{print $1, $3}' /etc/passwd           # Username and UID
awk -F: '{print $1 ":" $7}' /etc/passwd        # Username:Shell

# Conditionals
awk -F: '$3 >= 1000' /etc/passwd               # Users with UID >= 1000
awk '{if ($1 > 100) print}' file.txt

# Sum column
awk '{sum += $1} END {print sum}' numbers.txt

# Count lines
awk 'END {print NR}' file.txt                  # Same as wc -l

# Built-in variables
awk '{print NR, $0}' file.txt                  # Print line number
awk '{print NF, $0}' file.txt                  # Print field count

# Example: Memory usage by process
ps aux | awk '{print $4, $11}' | sort -rn | head -10

# Example: Parse disk usage
df -h | awk '$5 > 80 {print $1, $5}'           # Disks over 80% full
```

### CUT - Extract Columns:
```bash
# By delimiter
cut -d: -f1 /etc/passwd                        # First field (username)
cut -d: -f1,3,7 /etc/passwd                    # Multiple fields
cut -d: -f1-3 /etc/passwd                      # Range of fields

# By character position
cut -c1-10 file.txt                            # Characters 1-10
cut -c1,5,10 file.txt                          # Specific characters

# Example: Extract IPs from log
grep "Failed password" /var/log/secure | awk '{print $11}' | cut -d: -f1
```

### SORT and UNIQ:
```bash
# Sort alphabetically
sort file.txt

# Sort numerically
sort -n numbers.txt

# Sort reverse
sort -r file.txt

# Sort by column
sort -k2 file.txt                              # Sort by 2nd column
sort -t: -k3 -n /etc/passwd                    # By UID numerically

# Remove duplicates while sorting
sort -u file.txt

# Case insensitive
sort -f file.txt

# Uniq (remove adjacent duplicates)
sort file.txt | uniq
sort file.txt | uniq -c                        # Count occurrences
sort file.txt | uniq -d                        # Only show duplicates
sort file.txt | uniq -u                        # Only show unique lines
```

### Real-World Examples:
```bash
# Extract all IPs from log and count them
grep "Failed password" /var/log/secure | \
    awk '{print $11}' | \
    sort | uniq -c | sort -rn

# Top 10 commands in history
history | awk '{print $2}' | sort | uniq -c | sort -rn | head -10

# Find largest directories
du -h /var/* 2>/dev/null | sort -rh | head -10

# Users sorted by UID
awk -F: '{print $3, $1}' /etc/passwd | sort -n

# Extract emails from file
grep -Eo '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt

# Clean and sort list
cat list.txt | tr '[:upper:]' '[:lower:]' | sort -u > clean-list.txt
```

### Complex Pipeline:
```bash
# Analyze failed login attempts
cat /var/log/secure | \
    grep "Failed password" | \
    sed 's/.*from //' | \
    sed 's/ port.*//' | \
    sort | uniq -c | sort -rn | \
    awk '{print $2 " - " $1 " attempts"}' > failed-logins.txt
```

### Verification:
```bash
# Test your commands
grep "error" /var/log/messages | wc -l
awk -F: '$3 >= 1000 {print $1}' /etc/passwd | sort
sed -n '/^root/p' /etc/passwd
```
</details>

---

## 🎯 SCENARIO 8: FILE COMPRESSION & ARCHIVES (10 minutes)

### THE SITUATION:
You need to backup and compress various files and directories for offsite storage. Different compression methods offer different tradeoffs.

### YOUR TASKS:
1. Create tar archives (uncompressed, gzip, bzip2, xz)
2. Extract and list archive contents
3. Compress individual files
4. Create zip archives

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Create Test Files:
```bash
mkdir ~/archive_test
cd ~/archive_test
for i in {1..5}; do echo "Content of file $i" > file$i.txt; done
mkdir subdir
echo "Subdir content" > subdir/subfile.txt
```

### TAR Archives (Uncompressed):
```bash
# Create archive
tar -cvf archive.tar file*.txt subdir/
# c = create, v = verbose, f = file

# List contents
tar -tvf archive.tar
# t = list, v = verbose, f = file

# Extract
mkdir extract1
tar -xvf archive.tar -C extract1/
# x = extract, C = change to directory

# Append to existing archive
echo "New file" > file6.txt
tar -rvf archive.tar file6.txt
# r = append
```

### GZIP Compression:
```bash
# Create gzip compressed archive
tar -czf archive.tar.gz file*.txt subdir/
# z = gzip compression

# Alternative two-step:
tar -cf archive.tar file*.txt
gzip archive.tar                               # Creates archive.tar.gz

# Keep original
gzip -k archive.tar

# Extract
tar -xzf archive.tar.gz -C extract2/
# Or auto-detect:
tar -xf archive.tar.gz

# Compress individual file
gzip file1.txt                                 # Creates file1.txt.gz
gunzip file1.txt.gz                            # Decompress

# View compressed file
zcat file1.txt.gz
zless /var/log/messages*.gz
```

### BZIP2 Compression (Better compression, slower):
```bash
# Create bzip2 archive
tar -cjf archive.tar.bz2 file*.txt subdir/
# j = bzip2

# Extract
tar -xjf archive.tar.bz2 -C extract3/

# Compress individual file
bzip2 file2.txt                                # Creates file2.txt.bz2
bunzip2 file2.txt.bz2

# View compressed file
bzcat file2.txt.bz2
```

### XZ Compression (Best compression, slowest):
```bash
# Create xz archive
tar -cJf archive.tar.xz file*.txt subdir/
# J = xz

# Extract
tar -xJf archive.tar.xz -C extract4/

# Compress individual file
xz file3.txt                                   # Creates file3.txt.xz
unxz file3.txt.xz

# View compressed file
xzcat file3.txt.xz
```

### Auto-Detect Compression:
```bash
# Create with auto-compression (based on extension)
tar -caf archive.tar.gz file*.txt              # gzip
tar -caf archive.tar.bz2 file*.txt             # bzip2
tar -caf archive.tar.xz file*.txt              # xz

# Extract with auto-detection
tar -xf archive.tar.gz
tar -xf archive.tar.bz2
tar -xf archive.tar.xz
```

### ZIP Archives:
```bash
# Create zip archive
zip archive.zip file*.txt

# Add directory recursively
zip -r archive.zip subdir/

# List contents
unzip -l archive.zip

# Extract
unzip archive.zip -d extract5/

# Extract specific file
unzip archive.zip file1.txt

# Update existing archive
echo "Updated" > file1.txt
zip -u archive.zip file1.txt
```

### Comparison of Compression:
```bash
# Create test file
dd if=/dev/zero of=testfile bs=1M count=100

# Compare compression methods
time tar -czf test.tar.gz testfile
time tar -cjf test.tar.bz2 testfile
time tar -cJf test.tar.xz testfile

# Check sizes
ls -lh test.tar.*
```

### Real-World Backup Example:
```bash
# Full system backup excluding certain directories
sudo tar -czf /backup/system-backup-$(date +%Y%m%d).tar.gz \
    --exclude=/proc \
    --exclude=/sys \
    --exclude=/dev \
    --exclude=/tmp \
    --exclude=/backup \
    /

# Backup with progress
sudo tar -czf - /home | pv > /backup/home-backup.tar.gz

# Split large archives
tar -czf - /large/directory | split -b 1G - backup.tar.gz.part
# Reassemble:
# cat backup.tar.gz.part* | tar -xzf -
```

### Verification:
```bash
# Verify archive integrity
tar -tzf archive.tar.gz > /dev/null && echo "OK" || echo "CORRUPTED"
zip -T archive.zip
```
</details>

---

## 🎯 SCENARIO 9: HARD & SOFT LINKS (10 minutes)

### THE SITUATION:
You need to optimize storage by using hard links for duplicate files and create convenient access points using soft links (symlinks).

### YOUR TASKS:
1. Create hard links to save space for duplicate files
2. Create soft links for easy access to deeply nested directories
3. Understand the difference between hard and soft links
4. Verify link behavior when original file is deleted

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Understanding Links:
```bash
# HARD LINK: Points to the same inode (data) as original
# - Same file, different name
# - Changes to one affect the other
# - Deleting one doesn't affect the other
# - Can't cross filesystems
# - Can't link to directories

# SOFT LINK (SYMLINK): Points to the filename
# - Like Windows shortcut
# - If original deleted, link breaks
# - Can cross filesystems
# - Can link to directories
```

### Create Hard Links:
```bash
# Create original file
cd ~
echo "Important data" > original.txt
ls -li original.txt                            # Note inode number

# Create hard link
ln original.txt hardlink.txt

# Verify - same inode!
ls -li original.txt hardlink.txt
# Both files have same inode and link count = 2

# Check with stat
stat original.txt                              # Links: 2
stat hardlink.txt                              # Links: 2

# Modify via hard link
echo "More data" >> hardlink.txt
cat original.txt                               # Shows "More data" too!

# Delete original
rm original.txt
cat hardlink.txt                               # Still accessible!
```

### Real-World Hard Link Example:
```bash
# Save space for duplicate files
echo "Duplicate content" > file1.txt
cp file1.txt file2.txt                         # Wastes space

# Check disk usage
du -sh file1.txt file2.txt

# Use hard link instead
rm file2.txt
ln file1.txt file2.txt                         # No extra space used!

# Verify both files work
echo "Updated" >> file2.txt
cat file1.txt                                  # Shows "Updated"
```

### Create Soft Links (Symlinks):
```bash
# Absolute path symlink
mkdir -p /data/projects/important
echo "Project file" > /data/projects/important/file.txt

ln -s /data/projects/important ~/project-link

# Verify
ls -l ~/project-link                           # Shows arrow (->)
readlink ~/project-link                        # Shows target

# Access via symlink
cat ~/project-link/file.txt
cd ~/project-link
pwd -P                                         # Shows real path
pwd -L                                         # Shows logical path
```

### Relative Path Symlinks:
```bash
# Create directory structure
mkdir -p ~/docs/archived/2024
echo "Old document" > ~/docs/archived/2024/report.txt

# Create relative symlink
cd ~/docs
ln -s archived/2024/report.txt latest-report.txt

# Verify
ls -l latest-report.txt
cat latest-report.txt
```

### Symlinks for Directories:
```bash
# Link to /var/log for easy access
ln -s /var/log ~/logs

# Now you can:
cd ~/logs
ls -l
tail -f messages
```

### Broken Symlinks:
```bash
# Create symlink
echo "Original" > /tmp/original.txt
ln -s /tmp/original.txt ~/link-to-temp

# Verify it works
cat ~/link-to-temp

# Delete original
rm /tmp/original.txt

# Link is broken
ls -l ~/link-to-temp                           # Shows in red (if colors enabled)
cat ~/link-to-temp                             # Error: No such file

# Fix by recreating original
echo "New original" > /tmp/original.txt
cat ~/link-to-temp                             # Works again!
```

### Find and Manage Links:
```bash
# Find all hard links to a file
find ~ -samefile original.txt

# Find all symlinks
find ~ -type l

# Find broken symlinks
find ~ -type l ! -exec test -e {} \; -print

# Find hard links by inode
ls -li original.txt                            # Get inode number
find ~ -inum <inode-number>
```

### Practical Examples:
```bash
# Common use cases for symlinks:

# 1. Version management
ln -s /opt/app-v2.0 /opt/app-current
# Update just changes the symlink:
# ln -sfn /opt/app-v3.0 /opt/app-current

# 2. Configuration switching
ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/mysite

# 3. Easy access
ln -s /var/www/html ~/public_html

# 4. Shared libraries
ln -s /usr/lib/libcrypto.so.1.1 /usr/lib/libcrypto.so
```

### Comparing Links:
```bash
# Create test files
echo "Data" > original.txt

# Create both types of links
ln original.txt hard.txt                       # Hard link
ln -s original.txt soft.txt                    # Soft link

# Compare
ls -li original.txt hard.txt soft.txt
stat original.txt hard.txt soft.txt

# Delete original
rm original.txt

# Hard link still works!
cat hard.txt                                   # SUCCESS

# Soft link is broken
cat soft.txt                                   # ERROR: No such file
```

### Verification:
```bash
# Check links
ls -li                                         # Shows inodes and link counts
ls -l | grep '^l'                              # Shows only symlinks
find . -type l                                 # Find all symlinks
readlink -f ~/link                             # Show target of symlink
```
</details>

---

## 🎯 SCENARIO 10: NFS & AUTOFS (20 minutes)

### THE SITUATION:
Your organization needs centralized file storage. Set up NFS server to share directories and configure AutoFS for automatic mounting on clients.

### YOUR TASKS:
1. Configure NFS server and export a directory
2. Configure NFS client to mount the share
3. Make NFS mount persistent
4. Configure AutoFS for automatic on-demand mounting
5. Test both methods

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### NFS SERVER SETUP (VM2):
```bash
# Install NFS server
sudo yum install nfs-utils -y

# Create shared directory
sudo mkdir -p /nfs/shared
sudo chmod 755 /nfs/shared

# Create test file
echo "NFS test data" | sudo tee /nfs/shared/testfile.txt

# Configure exports
sudo vi /etc/exports
```

Add (replace with your network):
```
/nfs/shared  192.168.100.0/24(rw,sync,no_root_squash,no_subtree_check)

# Options explained:
# rw - Read/write
# sync - Synchronous writes
# no_root_squash - Root on client = root on server
# no_subtree_check - Improve performance
```

```bash
# Apply exports
sudo exportfs -avr
# -a = all, -v = verbose, -r = reexport

# Verify exports
sudo exportfs -v
cat /var/lib/nfs/etab

# Start NFS services
sudo systemctl enable --now nfs-server
sudo systemctl enable --now rpcbind
sudo systemctl status nfs-server

# Configure firewall
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-services
```

### NFS CLIENT SETUP (VM1):
```bash
# Install NFS client
sudo yum install nfs-utils -y

# Check NFS server exports
showmount -e 192.168.100.51
# Output: /nfs/shared 192.168.100.0/24

# Create mount point
sudo mkdir -p /mnt/nfs

# Mount temporarily
sudo mount -t nfs 192.168.100.51:/nfs/shared /mnt/nfs

# Verify
df -h | grep nfs
ls -l /mnt/nfs
cat /mnt/nfs/testfile.txt

# Test write access
echo "Client test" | sudo tee /mnt/nfs/clientfile.txt
```

### Make NFS Mount Persistent:
```bash
# Edit /etc/fstab
sudo vi /etc/fstab
```

Add:
```
192.168.100.51:/nfs/shared  /mnt/nfs  nfs  defaults,_netdev  0 0

# _netdev = wait for network before mounting
```

```bash
# Test fstab without rebooting
sudo umount /mnt/nfs
sudo mount -a
df -h | grep nfs

# Verify after reboot
sudo systemctl reboot
# After reboot:
df -h | grep nfs
```

### AUTOFS SETUP (Automatic On-Demand Mounting):
```bash
# Install AutoFS
sudo yum install autofs -y

# Configure master map
sudo vi /etc/auto.master
```

Add:
```
/shares  /etc/auto.nfs  --timeout=60
```

```bash
# Create NFS map file
sudo vi /etc/auto.nfs
```

Add:
```
# Key     Options                           Location
data      -rw,soft,intr,rsize=8192,wsize=8192  192.168.100.51:/nfs/shared
```

```bash
# Start AutoFS
sudo systemctl enable --now autofs
sudo systemctl status autofs
```

### Test AutoFS:
```bash
# Directory doesn't exist yet
ls /shares                                     # Empty or doesn't exist

# Access triggers mount
ls /shares/data                                # Now it appears!
cat /shares/data/testfile.txt

# Verify mounted
df -h | grep data
mount | grep data

# After timeout (60 seconds), automatically unmounts
sleep 65
df -h | grep data                              # Should be gone

# Access again - remounts automatically
cat /shares/data/testfile.txt
```

### Advanced AutoFS Configuration:
```bash
# Multiple NFS shares
sudo vi /etc/auto.nfs
```

Add:
```
data      -rw,soft    192.168.100.51:/nfs/shared
backup    -ro,soft    192.168.100.51:/nfs/backup
documents -rw,soft    192.168.100.51:/nfs/docs
```

```bash
# Wildcard matching
# In /etc/auto.master:
# /nfs  /etc/auto.nfs

# In /etc/auto.nfs:
# *  -rw,soft  192.168.100.51:/nfs/&
# This mounts /nfs/whatever as 192.168.100.51:/nfs/whatever
```

### Troubleshooting NFS:
```bash
# Server side:
sudo systemctl status nfs-server
sudo exportfs -v
sudo journalctl -u nfs-server
rpcinfo -p
sudo tail -f /var/log/messages

# Client side:
sudo systemctl status autofs
showmount -e 192.168.100.51
sudo journalctl -u autofs
mount | grep nfs

# Test manual mount
sudo mount -t nfs -v 192.168.100.51:/nfs/shared /mnt/test

# Debug AutoFS
sudo automount -f -v                           # Run in foreground
```

### NFS Performance Options:
```bash
# For better performance, tune mount options:
# In /etc/fstab or AutoFS:
# rsize=32768,wsize=32768,hard,intr,tcp
```

### Verification:
```bash
# Server:
sudo exportfs -v
systemctl status nfs-server
showmount -e localhost

# Client:
showmount -e 192.168.100.51
df -h | grep nfs
cat /shares/data/testfile.txt
systemctl status autofs
```
</details>

---

## 🎯 SCENARIO 11: SYSTEM LOGGING (10 minutes)

### THE SITUATION:
You need to review system logs for troubleshooting, make journals persistent, and monitor specific services.

### YOUR TASKS:
1. View systemd journal logs with various filters
2. Make journal logs persistent across reboots
3. Check disk usage of journals
4. Monitor logs in real-time
5. Export logs for analysis

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Basic Journalctl Usage:
```bash
# View all logs
sudo journalctl

# View logs and jump to end
sudo journalctl -e

# Follow logs in real-time
sudo journalctl -f
# Press Ctrl+C to stop

# View boot logs
sudo journalctl -b                             # Current boot
sudo journalctl -b -1                          # Previous boot
sudo journalctl --list-boots                   # List all boots
```

### Filter by Service:
```bash
# View SSH daemon logs
sudo journalctl -u sshd

# View multiple services
sudo journalctl -u sshd -u httpd

# View custom service
sudo journalctl -u monitor.service

# Follow service logs
sudo journalctl -u httpd -f
```

### Filter by Time:
```bash
# Since specific date
sudo journalctl --since "2025-01-01"
sudo journalctl --since "2025-01-01 10:00:00"

# Time range
sudo journalctl --since "2025-01-01" --until "2025-01-02"

# Relative time
sudo journalctl --since today
sudo journalctl --since yesterday
sudo journalctl --since "1 hour ago"
sudo journalctl --since "30 minutes ago"

# Last N lines
sudo journalctl -n 50                          # Last 50 lines
sudo journalctl -u sshd -n 20                  # Last 20 lines of sshd
```

### Filter by Priority:
```bash
# View by priority level
sudo journalctl -p err                         # Errors only
sudo journalctl -p warning                     # Warnings and above
sudo journalctl -p info                        # Info and above

# Priority levels:
# 0: emerg (emergency)
# 1: alert
# 2: crit (critical)
# 3: err (error)
# 4: warning
# 5: notice
# 6: info
# 7: debug

# Errors today
sudo journalctl -p err --since today
```

### Filter by Specific Command:
```bash
# Logs from specific executable
sudo journalctl /usr/bin/sudo
sudo journalctl /usr/sbin/sshd

# Find command path first
which sudo
sudo journalctl $(which sudo)
```

### Make Journals Persistent:
```bash
# Check if persistent storage exists
ls -ld /var/log/journal

# Create directory if missing
sudo mkdir -p /var/log/journal

# Set ownership
sudo systemd-tmpfiles --create --prefix /var/log/journal

# Restart journald
sudo systemctl restart systemd-journald

# Verify persistent storage
sudo journalctl --verify
```

### Configure Journal Settings:
```bash
# Edit configuration
sudo vi /etc/systemd/journald.conf
```

Modify:
```
[Journal]
Storage=persistent
#SystemMaxUse=500M
#SystemKeepFree=1G
#SystemMaxFileSize=100M
#MaxRetentionSec=1month
```

```bash
# Apply changes
sudo systemctl restart systemd-journald
```

### Check Journal Disk Usage:
```bash
# View disk usage
sudo journalctl --disk-usage

# Verify journal files
sudo journalctl --verify

# Vacuum old journals
sudo journalctl --vacuum-time=30d              # Keep last 30 days
sudo journalctl --vacuum-size=500M             # Keep only 500MB

# Rotate journals
sudo journalctl --rotate
```

### Export Logs:
```bash
# Export to file
sudo journalctl --since today > today.log
sudo journalctl -u httpd --since "1 hour ago" > httpd-last-hour.log

# Export in JSON format
sudo journalctl -u sshd -o json > sshd.json

# Export in short format
sudo journalctl -u httpd -o short > httpd-short.log

# Available formats:
# short, short-iso, short-precise, short-monotonic
# verbose, export, json, json-pretty, json-sse, cat
```

### Search Logs:
```bash
# Grep in logs
sudo journalctl | grep "failed"
sudo journalctl | grep -i "error"

# Case-insensitive search
sudo journalctl -g "failed"                    # Built-in grep
sudo journalctl -g "failed|error"              # Multiple patterns

# Combine with other filters
sudo journalctl -u sshd -g "Failed password"
```

### View Kernel Messages:
```bash
# Kernel messages only
sudo journalctl -k
sudo journalctl -k -b                          # Kernel logs this boot

# Traditional dmesg
dmesg | tail -50
dmesg | grep -i error
```

### Monitor Specific User:
```bash
# Logs for specific user
sudo journalctl _UID=1000

# Get UID
id username
```

### Real-Time Monitoring:
```bash
# Monitor all logs
sudo journalctl -f

# Monitor specific service
sudo journalctl -u httpd -f

# Monitor with filters
sudo journalctl -u sshd -f -p warning

# Monitor multiple units
sudo journalctl -u sshd -u httpd -f
```

### Verification:
```bash
# Check journal status
sudo systemctl status systemd-journald

# Check persistent storage
ls -lh /var/log/journal/

# Check disk usage
sudo journalctl --disk-usage

# Verify integrity
sudo journalctl --verify
```
</details>

---



## ✅ DRILL B COMPLETION CHECKLIST

- [ ] Apache service installed and enabled
- [ ] Custom monitor service created and running
- [ ] atd service masked
- [ ] Cron jobs configured for backups and maintenance
- [ ] at job scheduled
- [ ] Process management commands practiced
- [ ] tuned profile changed
- [ ] GRUB configuration modified
- [ ] Multiple bash scripts created and tested
- [ ] Text processing pipelines working
- [ ] Archives created with tar, gzip, bzip2, xz
- [ ] Hard and soft links created
- [ ] NFS server and client configured
- [ ] AutoFS working correctly
- [ ] Journal logs persistent
- [ ] Container running with systemd
- [ ] Custom YUM repository added
- [ ] System rebooted and all services persist

**Time Completed: _____ minutes**

---

## 💪 MASTERY INDICATORS

You've mastered Drill B when you can:
1. Complete all tasks in < 90 minutes
2. All services and configurations persist after reboot
3. Write bash scripts from memory
4. Troubleshoot service issues quickly
5. Explain every command and option used
