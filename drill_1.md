# RHCSA DRILL A - Storage, Network & Security
**Time Target: 90-120 minutes | Days 1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21, 23, 25, 27, 29**

---

## DRILL A PRACTICE FLOW

- [ ] Break Root Password
- [ ] Configure Network
- [ ] Add/Update Repository
- [ ] User and Groups
- [ ] MBR & GPT Partitions
- [ ] PV -> VG -> LV -> FS -> Mount -> fstab
- [ ] Extend VG and LV (ext4, xfx)
- [ ] LV -> SWAP -> fstab
- [ ] /swapfile -> fstab
- [ ] Container
- [ ] Firewall
- [ ] SSH
- [ ] Selinux

---

## 🎯 SCENARIO 1: SYSTEM RECOVERY (15 minutes)

### THE SITUATION:
You've just been hired as a system administrator at TechCorp. Your predecessor left without documenting the root password for the production server `server1.techcorp.com`. Management needs immediate access to perform critical maintenance.

### YOUR TASKS:
1. Reset the root password to `RedHat2025!`
2. Ensure SELinux contexts are properly restored after the reset
3. Configure the system to boot into multi-user (text) mode by default
4. Verify you can switch to graphical mode temporarily without rebooting

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Reset Root Password (RHEL 8 Method):
```bash
# At GRUB menu: Press 'e' to edit
# Navigate to line starting with 'linux'
# Press Ctrl+E to go to end of line
# Add: rd.break
# Press Ctrl+X to boot

# At switch_root prompt:
mount | grep /sysroot                          # Verify read-only mount
mount -o remount,rw /sysroot                   # Remount as read-write
chroot /sysroot                                # Change root to actual filesystem

passwd root                                    # Enter: RedHat2025!
touch /.autorelabel                            # CRITICAL: Fix SELinux contexts

exit                                           # Exit chroot
exit                                           # Exit emergency shell (system reboots)
```

### Reset Root Password (RHEL 9 Method):
```bash
# At GRUB menu: Press 'e' to edit
# Navigate to line starting with 'linux'
# Change 'ro' to 'rw'
# Add at end: init=/bin/bash
# Press Ctrl+X to boot

passwd root                                    # Enter: RedHat2025!
touch /.autorelabel                            # Fix SELinux contexts
exec /sbin/init                                # Continue boot process
```

### Configure Boot Target:
```bash
# After logging in as root:
systemctl get-default                          # Check current (should be graphical.target)

sudo systemctl set-default multi-user.target  # Set to text mode
# Output: Removed /etc/systemd/system/default.target
#         Created symlink...

# Reboot to verify
sudo systemctl reboot

# After reboot, temporarily switch to graphical:
sudo systemctl isolate graphical.target
```

### Verification:
```bash
systemctl get-default                          # Should show: multi-user.target
```
</details>

---

## 🎯 SCENARIO 2: DISK PARTITIONING FOR NEW SERVER (20 minutes)

### THE SITUATION:
You're setting up a new server that has 4 additional hard drives attached (`/dev/vdb`, `/dev/vdc`, `/dev/vdd`, `/dev/vde`). The server needs both legacy BIOS and modern UEFI support for different deployment scenarios.

### YOUR TASKS:
1. On `/dev/vdb` (10GB), create an MBR partition table with:
   - Partition 1: 4GB primary partition
   - Partition 2: 4GB primary partition (type: Linux swap)
   - Partition 3: Extended partition using remaining space
   - Partition 4: 2GB logical partition (type: Linux LVM)

2. On `/dev/vdc` (10GB), create a GPT partition table with:
   - Partition 1: 5GB partition
   - Partition 2: 5GB partition (set LVM flag)

3. Verify all partitions are recognized by the kernel

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### MBR Partitioning with fdisk:
```bash
# Check available disks
lsblk

# Start fdisk on /dev/vdb
sudo fdisk /dev/vdb

# Inside fdisk interactive mode:
Command: n                                     # New partition
Select: p                                      # Primary
Partition number: 1                            # First partition
First sector: [Enter]                          # Default
Last sector: +4G                               # 4GB size

Command: n                                     # Second partition
Select: p
Partition number: 2
First sector: [Enter]
Last sector: +4G

Command: t                                     # Change type
Partition number: 2
Hex code: 82                                   # Linux swap

Command: n                                     # Extended partition
Select: e                                      # Extended
Partition number: 3
First sector: [Enter]
Last sector: [Enter]                           # Use remaining space

Command: n                                     # Logical partition
Partition number: [Enter]                      # Auto-assigns
First sector: [Enter]
Last sector: +2G

Command: t                                     # Change type
Partition number: 4
Hex code: 8e                                   # Linux LVM

Command: p                                     # Print to verify
Command: w                                     # Write changes

# Inform kernel of partition changes
sudo partprobe /dev/vdb

# Verify
lsblk /dev/vdb
```

### GPT Partitioning with parted:
```bash
# Start parted on /dev/vdc
sudo parted /dev/vdc

# Inside parted:
(parted) print                                 # Check current state
(parted) mklabel gpt                           # Create GPT partition table

(parted) mkpart primary 1MiB 5GiB              # First partition
(parted) mkpart primary 5GiB 10GiB             # Second partition

(parted) set 2 lvm on                          # Set LVM flag on partition 2

(parted) print                                 # Verify partitions
(parted) quit

# Verify
lsblk /dev/vdc
```

### Final Verification:
```bash
lsblk                                          # View all partitions
sudo fdisk -l /dev/vdb                         # Detailed MBR info
sudo parted /dev/vdc print                     # Detailed GPT info
```
</details>

---

## 🎯 SCENARIO 3: ENTERPRISE STORAGE WITH LVM (25 minutes)

### THE SITUATION:
Your company needs a flexible storage solution for their database server. The partitions you created need to be configured with LVM to allow future expansion. The database team requires 8GB of storage now but expects to need more soon.

### YOUR TASKS:
1. Create physical volumes from `/dev/vdb4` and `/dev/vdc2`
2. Create a volume group called `datavg` using both physical volumes
3. Create two logical volumes:
   - `datalv`: 5GB, formatted with XFS filesystem
   - `backuplv`: 3GB, formatted with ext4 filesystem
4. Mount `datalv` on `/data` and `backuplv` on `/backup`
5. Make both mounts persistent using UUID in `/etc/fstab`
6. The database is growing! Extend `datalv` by 2GB
7. The backup system needs adjustment - reduce `backuplv` to 2GB (ext4 only supports shrinking)

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Create Physical Volumes:
```bash
# Create PVs
sudo pvcreate /dev/vdb4 /dev/vdc2

# Verify
sudo pvs
sudo pvdisplay                                 # Detailed view
```

### Create Volume Group:
```bash
# Create VG named 'datavg'
sudo vgcreate datavg /dev/vdb4 /dev/vdc2

# Verify
sudo vgs
sudo vgdisplay datavg                          # Detailed view
```

### Create Logical Volumes:
```bash
# Create datalv (5GB)
sudo lvcreate -L 5G -n datalv datavg

# Create backuplv (3GB)
sudo lvcreate -L 3G -n backuplv datavg

# Verify
sudo lvs
sudo lvdisplay /dev/datavg/datalv              # Detailed view
```

### Format Filesystems:
```bash
# Format datalv with XFS
sudo mkfs.xfs /dev/datavg/datalv

# Format backuplv with ext4
sudo mkfs.ext4 /dev/datavg/backuplv

# Verify
lsblk -f | grep datavg
```

### Create Mount Points and Mount:
```bash
# Create directories
sudo mkdir -p /data /backup

# Mount temporarily
sudo mount /dev/datavg/datalv /data
sudo mount /dev/datavg/backuplv /backup

# Verify
df -h | grep -E 'data|backup'
```

### Make Persistent in /etc/fstab:
```bash
# Get UUIDs
sudo blkid /dev/datavg/datalv
sudo blkid /dev/datavg/backuplv

# Edit fstab
sudo vi /etc/fstab

# Add these lines (replace UUIDs with your actual values):
# UUID=xxxx-xxxx-xxxx  /data    xfs   defaults  0 0
# UUID=yyyy-yyyy-yyyy  /backup  ext4  defaults  0 0

# TEST before rebooting (CRITICAL!)
sudo umount /data /backup
sudo mount -a                                   # Mount all from fstab
df -h | grep -E 'data|backup'                  # Verify mounts worked
```

### Extend datalv (Database Growth):
```bash
# Extend LV by 2GB
sudo lvextend -L +2G /dev/datavg/datalv

# Extend XFS filesystem (MUST do this for XFS)
sudo xfs_growfs /data

# Verify
df -h /data                                    # Should show 7GB now
sudo lvs | grep datalv
```

### Reduce backuplv (Backup Adjustment):
```bash
# WARNING: This only works with ext4, not XFS!
# ALWAYS backup data first in production!

# Unmount first
sudo umount /backup

# Check filesystem integrity
sudo e2fsck -f /dev/datavg/backuplv

# Resize filesystem to 2GB FIRST
sudo resize2fs /dev/datavg/backuplv 2G

# Then reduce LV to match
sudo lvreduce -L 2G /dev/datavg/backuplv
# Confirm with 'y'

# Remount
sudo mount /backup

# Verify
df -h /backup                                  # Should show ~2GB
sudo lvs | grep backuplv
```

### Final Verification:
```bash
# Check everything
sudo pvs
sudo vgs
sudo lvs
df -h
lsblk

# TEST REBOOT PERSISTENCE
sudo systemctl reboot
# After reboot:
df -h | grep -E 'data|backup'                  # Should auto-mount
```
</details>

---

## 🎯 SCENARIO 4: SWAP SPACE CONFIGURATION (10 minutes)

### THE SITUATION:
The server is experiencing memory pressure during peak hours. System administrators need to add swap space to prevent out-of-memory errors. You have `/dev/vdb2` (marked as swap type) available, and you need additional swap as a file.

### YOUR TASKS:
1. Configure `/dev/vdb2` as swap partition (2GB)
2. Create a 2GB swap file at `/swapfile`
3. Make both swap spaces persistent across reboots
4. Verify total swap space is 4GB

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Configure Swap Partition:
```bash
# Format as swap (partition already type 82 from earlier)
sudo mkswap /dev/vdb2

# Enable swap
sudo swapon /dev/vdb2

# Verify
swapon --show
free -h                                        # Check total swap
```

### Create Swap File:
```bash
# Create 2GB file
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress

# Secure permissions (CRITICAL!)
sudo chmod 600 /swapfile

# Format as swap
sudo mkswap /swapfile

# Enable swap
sudo swapon /swapfile

# Verify
swapon --show
free -h                                        # Should show ~4GB total swap
```

### Make Persistent:
```bash
# Edit /etc/fstab
sudo vi /etc/fstab

# Add these lines:
# /dev/vdb2     none    swap    defaults    0 0
# /swapfile     none    swap    defaults    0 0

# TEST without rebooting
sudo swapoff -a                                # Turn off all swap
swapon --show                                  # Should be empty

sudo swapon -a                                 # Enable all swap from fstab
swapon --show                                  # Should show both swaps
free -h                                        # Verify total swap
```

### Final Verification After Reboot:
```bash
sudo systemctl reboot
# After reboot:
swapon --show
free -h
```
</details>

---

## 🎯 SCENARIO 5: RAID STORAGE CONFIGURATION (20 minutes)

### THE SITUATION:
Your organization requires high-performance, fault-tolerant storage for critical applications. You have `/dev/vdd` and `/dev/vde` available. You need to create a RAID1 (mirrored) array for redundancy and a RAID5 array (if you had 3+ disks, but we'll demonstrate RAID concepts).

### YOUR TASKS:
1. Create a RAID1 array using `/dev/vdd` and `/dev/vde` named `/dev/md0`
2. Format the RAID array with ext4
3. Mount it on `/raid`
4. Make the mount persistent
5. Verify RAID status and test one disk failure simulation

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Install RAID Tools:
```bash
sudo yum install mdadm -y
```

### Create RAID1 Array:
```bash
# Create RAID1 mirror with 2 devices
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdd /dev/vde

# Answer 'y' to confirm

# Watch array creation (optional)
cat /proc/mdstat

# Detailed status
sudo mdadm --detail /dev/md0
```

### Format and Mount:
```bash
# Format with ext4
sudo mkfs.ext4 /dev/md0

# Create mount point
sudo mkdir /raid

# Mount
sudo mount /dev/md0 /raid

# Verify
df -h | grep raid
```

### Make Persistent:
```bash
# Get UUID
sudo blkid /dev/md0

# Edit fstab
sudo vi /etc/fstab

# Add (replace UUID with actual):
# UUID=xxxx-xxxx  /raid  ext4  defaults  0 0

# Save RAID configuration
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf

# TEST
sudo umount /raid
sudo mount -a
df -h | grep raid
```

### Test RAID Functionality:
```bash
# Create test file
echo "RAID test data" | sudo tee /raid/testfile.txt
cat /raid/testfile.txt

# Check RAID status
sudo mdadm --detail /dev/md0
cat /proc/mdstat

# Simulate disk failure (CAREFUL - for testing only!)
sudo mdadm --manage /dev/md0 --fail /dev/vde

# Check status - should show degraded array
sudo mdadm --detail /dev/md0
cat /proc/mdstat

# Data should still be accessible
cat /raid/testfile.txt

# Remove failed disk
sudo mdadm --manage /dev/md0 --remove /dev/vde

# Re-add disk (simulate replacement)
sudo mdadm --manage /dev/md0 --add /dev/vde

# Watch rebuild
watch cat /proc/mdstat                         # Ctrl+C to exit

# Verify healthy status
sudo mdadm --detail /dev/md0
```

### Understanding RAID Levels:
```bash
# RAID 0: Striping (performance, no redundancy)
# sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/vdd /dev/vde

# RAID 1: Mirroring (redundancy, we did this)
# sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdd /dev/vde

# RAID 5: Striping with parity (needs 3+ disks)
# sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/vdd /dev/vde /dev/vdf

# RAID 6: Like RAID5 but 2 parity blocks (needs 4+ disks)

# RAID 10: Mirrored stripe (needs 4+ disks)
```

### Verification:
```bash
lsblk
cat /proc/mdstat
sudo mdadm --detail /dev/md0
df -h /raid
```
</details>

---

## 🎯 SCENARIO 6: NETWORK CONFIGURATION (20 minutes)

### THE SITUATION:
Your server needs to be configured with static IP addressing for production use. The network team has provided you with: IP: 192.168.100.50/24, Gateway: 192.168.100.1, DNS: 8.8.8.8, 8.8.4.4. The server also needs IPv6 support and a proper hostname.

### YOUR TASKS:
1. Configure static IPv4: 192.168.100.50/24, gateway 192.168.100.1, DNS 8.8.8.8 and 8.8.4.4
2. Configure static IPv6: 2001:db8::50/64, gateway 2001:db8::1
3. Set hostname to `prod-server.example.com`
4. Add hostname to `/etc/hosts`
5. Ensure NetworkManager starts at boot
6. Verify connectivity to internet

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Check Current Configuration:
```bash
ip link show                                   # Show interfaces
ip addr show                                   # Show IP addresses
ip route show                                  # Show routes
cat /etc/resolv.conf                           # Show DNS
```

### Configure Static IPv4:
```bash
# List connections
nmcli con show

# Configure (replace "System eth0" with your connection name)
CONN_NAME="System eth0"

sudo nmcli con mod "$CONN_NAME" ipv4.addresses 192.168.100.50/24
sudo nmcli con mod "$CONN_NAME" ipv4.gateway 192.168.100.1
sudo nmcli con mod "$CONN_NAME" ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli con mod "$CONN_NAME" ipv4.method manual

# Apply changes
sudo nmcli con up "$CONN_NAME"

# Verify
ip addr show
ip route show
cat /etc/resolv.conf
```

### Configure Static IPv6:
```bash
sudo nmcli con mod "$CONN_NAME" ipv6.addresses 2001:db8::50/64
sudo nmcli con mod "$CONN_NAME" ipv6.gateway 2001:db8::1
sudo nmcli con mod "$CONN_NAME" ipv6.method manual

# Apply changes
sudo nmcli con up "$CONN_NAME"

# Verify
ip -6 addr show
ip -6 route show
```

### Set Hostname:
```bash
# Set hostname
sudo hostnamectl set-hostname prod-server.example.com

# Verify
hostnamectl status
hostname
hostname -f                                    # Fully qualified domain name
```

### Configure /etc/hosts:
```bash
# Edit hosts file
sudo vi /etc/hosts

# Add these lines:
# 192.168.100.50    prod-server.example.com prod-server
# 2001:db8::50      prod-server.example.com prod-server

# Verify
ping -c 2 prod-server
cat /etc/hosts
```

### Ensure NetworkManager at Boot:
```bash
# Check status
sudo systemctl status NetworkManager

# Enable if not already
sudo systemctl enable NetworkManager
sudo systemctl is-enabled NetworkManager       # Should show "enabled"
```

### Test Connectivity:
```bash
# IPv4 connectivity
ping -c 3 8.8.8.8                              # Ping Google DNS
ping -c 3 google.com                           # Test DNS resolution

# IPv6 connectivity (if available)
ping6 -c 3 ::1                                 # Ping localhost
ping6 -c 3 2001:4860:4860::8888                # Ping Google DNS IPv6

# Check DNS
nslookup google.com
dig google.com
```

### Alternative: Using nmtui:
```bash
# Text user interface (easier for some)
sudo nmtui

# Navigate with arrow keys:
# 1. Choose "Edit a connection"
# 2. Select your connection
# 3. Modify IPv4/IPv6 settings
# 4. Set to "Manual" and enter IPs
# 5. OK to save

# Apply changes
sudo nmcli con reload
sudo nmcli con up "$CONN_NAME"
```

### Verification:
```bash
# Complete verification
ip addr show
ip route show
ip -6 route show
cat /etc/resolv.conf
hostnamectl
ping -c 2 google.com
```
</details>

---

## 🎯 SCENARIO 7: FIREWALL CONFIGURATION (15 minutes)

### THE SITUATION:
Your server hosts a web application on port 80/443 and an application server on port 8080. SSH is running on custom port 2222. You need to configure firewalld to allow necessary traffic while keeping the server secure.

### YOUR TASKS:
1. Allow HTTP (80), HTTPS (443), and custom SSH port (2222)
2. Allow port 8080 for application server
3. Create a trusted zone for internal network 10.0.0.0/24
4. Block all other incoming traffic (default behavior)
5. Make all rules persistent

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Check Firewall Status:
```bash
# Ensure firewalld is running
sudo systemctl status firewalld
sudo systemctl enable --now firewalld

# Check current configuration
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

### Allow HTTP and HTTPS:
```bash
# Add services (temporary)
sudo firewall-cmd --add-service=http
sudo firewall-cmd --add-service=https

# Verify (temporary)
sudo firewall-cmd --list-services

# Make permanent
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
```

### Allow Custom SSH Port 2222:
```bash
# Remove default SSH service
sudo firewall-cmd --remove-service=ssh
sudo firewall-cmd --permanent --remove-service=ssh

# Add custom SSH port
sudo firewall-cmd --add-port=2222/tcp
sudo firewall-cmd --permanent --add-port=2222/tcp
```

### Allow Application Port 8080:
```bash
# Add custom port
sudo firewall-cmd --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
```

### Reload to Apply All Permanent Rules:
```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### Configure Trusted Zone for Internal Network:
```bash
# Add source network to trusted zone
sudo firewall-cmd --zone=trusted --add-source=10.0.0.0/24
sudo firewall-cmd --permanent --zone=trusted --add-source=10.0.0.0/24

# Reload
sudo firewall-cmd --reload

# Verify zones
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=trusted --list-all
sudo firewall-cmd --zone=public --list-all
```

### Advanced: Rich Rules (Optional):
```bash
# Allow SSH only from specific IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" port port="2222" protocol="tcp" accept'

# Block specific IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.99" reject'

# Reload
sudo firewall-cmd --reload
```

### Test Firewall:
```bash
# From another machine (or use telnet/nc if available):
# ssh -p 2222 user@192.168.100.50
# curl http://192.168.100.50
# curl http://192.168.100.50:8080

# From server, test locally:
sudo ss -tulpn | grep -E ':80|:443|:2222|:8080'
```

### Verification:
```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --zone=trusted --list-all
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all-zones
```
</details>

---

## 🎯 SCENARIO 8: SSH SECURITY HARDENING (15 minutes)

### THE SITUATION:
Security audit requires you to harden SSH access: disable root login, move SSH to port 2222, disable password authentication, and implement key-based authentication only.

### YOUR TASKS:
1. Generate SSH key pair (RSA, 4096 bits)
2. Configure SSH to run on port 2222
3. Disable root login
4. Disable password authentication
5. Configure key-based authentication
6. Update SELinux for port 2222
7. Test secure connection

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Generate SSH Key Pair:
```bash
# Generate key
ssh-keygen -t rsa -b 4096 -C "admin@prod-server.example.com"

# Press Enter for default location (/home/user/.ssh/id_rsa)
# Enter passphrase (optional but recommended)

# Verify
ls -la ~/.ssh/
cat ~/.ssh/id_rsa.pub                          # Public key
```

### Copy Public Key to Remote Server:
```bash
# If you're setting up for another server:
ssh-copy-id -p 22 user@remote-server           # Use current port

# Or manually:
# cat ~/.ssh/id_rsa.pub | ssh user@remote "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Set correct permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

### Configure SSH Server:
```bash
# Backup original config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Edit SSH config
sudo vi /etc/ssh/sshd_config
```

Make these changes:
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```

### Update SELinux for Port 2222:
```bash
# Add port to SELinux policy
sudo semanage port -a -t ssh_port_t -p tcp 2222

# Verify
sudo semanage port -l | grep ssh
```

### Reload SSH Service:
```bash
# Test configuration
sudo sshd -t                                   # Should show no errors

# Reload service
sudo systemctl reload sshd

# Verify service status
sudo systemctl status sshd
sudo ss -tulpn | grep 2222                     # Verify listening on port 2222
```

### Test Connection:
```bash
# From another terminal (BEFORE closing current session!):
ssh -p 2222 username@localhost

# Test key authentication
ssh -p 2222 -i ~/.ssh/id_rsa username@192.168.100.50

# Test that root login is denied
ssh -p 2222 root@localhost                     # Should be refused

# Test that password auth is disabled
# (Try from machine without key - should fail)
```

### Create SSH Client Config (Optional but useful):
```bash
# Make config easier
vi ~/.ssh/config
```

Add:
```
Host prod-server
    HostName prod-server.example.com
    Port 2222
    User yourusername
    IdentityFile ~/.ssh/id_rsa
```

```bash
# Now you can simply:
ssh prod-server
```

### Secure Copy Files:
```bash
# SCP examples with custom port
scp -P 2222 localfile.txt user@prod-server:/tmp/
scp -P 2222 user@prod-server:/remote/file.txt ./

# Rsync examples
rsync -avz -e "ssh -p 2222" /local/dir/ user@prod-server:/remote/dir/
```

### Verification:
```bash
sudo systemctl status sshd
sudo ss -tulpn | grep 2222
sudo semanage port -l | grep 2222
sudo firewall-cmd --list-ports | grep 2222
cat /etc/ssh/sshd_config | grep -E "Port|PermitRootLogin|PasswordAuthentication"
```
</details>

---

## 🎯 SCENARIO 9: SELINUX CONFIGURATION (20 minutes)

### THE SITUATION:
Your web server is having permission issues. Apache can't access files in `/web`, and you need to configure SELinux properly. The application also needs to connect to a database server, but SELinux is blocking network connections.

### YOUR TASKS:
1. Verify SELinux is in enforcing mode
2. Create `/web` directory for Apache content
3. Fix SELinux context for `/web`
4. Allow Apache to make network connections (boolean)
5. Configure Apache to run on custom port 8080 (SELinux port label)
6. Diagnose and fix any AVC denials

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Check SELinux Status:
```bash
# Current mode
getenforce                                     # Should show: Enforcing

# Detailed status
sestatus

# If not enforcing, set it:
sudo setenforce 1                              # Temporary
getenforce

# For permanent:
sudo vi /etc/selinux/config
# Set: SELINUX=enforcing
```

### Create Web Directory with Wrong Context:
```bash
# Create directory
sudo mkdir /web
sudo mkdir /web/html

# Create test file
echo "<h1>Test Page</h1>" | sudo tee /web/html/index.html

# Check context (will be wrong!)
ls -Z /web
ls -Z /web/html/
# Shows: unconfined_u:object_r:default_t:s0
# Should be: httpd_sys_content_t
```

### Fix SELinux File Context:
```bash
# Add context rule
sudo semanage fcontext -a -t httpd_sys_content_t '/web(/.*)?'

# Apply context
sudo restorecon -Rv /web

# Verify
ls -Z /web
ls -Z /web/html/
# Now shows: httpd_sys_content_t
```

### View Current Contexts:
```bash
# View file contexts
ls -Z /var/www/html                            # Compare with default
ls -Z /web/html                                # Your custom location

# View process contexts
ps -eZ | grep httpd

# View your user context
id -Z
```

### Allow Apache Network Connections:
```bash
# Check current boolean value
getsebool httpd_can_network_connect

# Enable boolean (temporary)
sudo setsebool httpd_can_network_connect on

# Make permanent
sudo setsebool -P httpd_can_network_connect on

# Verify
getsebool httpd_can_network_connect            # Should be: on

# List all httpd booleans
getsebool -a | grep httpd
```

### Configure SELinux Port Label for Port 8080:
```bash
# View current HTTP ports
sudo semanage port -l | grep http_port_t

# Add port 8080
sudo semanage port -a -t http_port_t -p tcp 8080

# Verify
sudo semanage port -l | grep http_port_t
# Should show: 8080 in the list

# If you need to remove it:
# sudo semanage port -d -t http_port_t -p tcp 8080
```

### Diagnose SELinux Denials:
```bash
# Install troubleshooting tools if needed
sudo yum install setroubleshoot-server -y

# View recent denials
sudo ausearch -m avc -ts recent

# Get detailed information
sudo sealert -a /var/log/audit/audit.log

# View with journalctl
sudo journalctl -t setroubleshoot --since today

# Real-time monitoring
sudo tail -f /var/log/audit/audit.log | grep denied
```

### Create Test Apache Config (if installed):
```bash
# Install Apache if needed
sudo yum install httpd -y

# Edit Apache config
sudo vi /etc/httpd/conf/httpd.conf
# Change: Listen 8080
# Change: DocumentRoot "/web/html"

# Update directory permissions in config
```

Add to httpd.conf:
```apache
<Directory "/web/html">
    AllowOverride None
    Require all granted
</Directory>
```

```bash
# Start Apache
sudo systemctl start httpd
sudo systemctl enable httpd

# Check status
sudo systemctl status httpd

# Test
curl http://localhost:8080
```

### Troubleshoot Common Issues:
```bash
# If Apache won't start, check:
sudo journalctl -xe -u httpd
sudo ausearch -m avc -ts recent

# Generate policy module from denials (if needed)
sudo ausearch -c 'httpd' --raw | audit2allow -M my-httpd
sudo semodule -X 300 -i my-httpd.pp
```

### Advanced: Modify Existing Context:
```bash
# Change context of specific file
sudo chcon -t httpd_sys_content_t /web/html/special.html

# Restore all files to policy default
sudo restorecon -Rv /var/www/html
```

### Verification:
```bash
getenforce
sestatus
ls -Z /web/html/
getsebool httpd_can_network_connect
sudo semanage port -l | grep 8080
sudo semanage fcontext -l | grep '/web'
```
</details>

---

## 🎯 SCENARIO 10: CONTAINER MANAGEMENT (15 minutes)

### THE SITUATION:
Deploy a containerized application using Podman, configure persistent storage, and set it up as a systemd service for automatic startup.

### YOUR TASKS:
1. Install Podman
2. Pull and run an nginx container
3. Configure port mapping and persistent volume
4. Create systemd service for rootless container
5. Test persistence across reboots

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### Install Podman:
```bash
# Install Podman
sudo yum install podman -y

# Verify installation
podman --version
podman info
```

### Search and Pull Images:
```bash
# Search for images
podman search nginx

# Pull nginx image
podman pull docker.io/library/nginx:latest

# List local images
podman images
```

### Run Basic Container:
```bash
# Run nginx in detached mode
podman run -d --name webserver -p 8080:80 nginx

# Verify container is running
podman ps
podman ps -a                                   # Include stopped containers

# Check logs
podman logs webserver
podman logs -f webserver                       # Follow logs

# Test the web server
curl http://localhost:8080
```

### Container Management:
```bash
# Stop container
podman stop webserver

# Start container
podman start webserver

# Restart container
podman restart webserver

# Pause/unpause container
podman pause webserver
podman unpause webserver

# Execute command in running container
podman exec webserver ls /usr/share/nginx/html
podman exec -it webserver /bin/bash            # Interactive shell
# Inside container: ls, cat index.html, exit

# Remove container
podman stop webserver
podman rm webserver

# Remove image
podman rmi nginx
```

### Persistent Storage:
```bash
# Create local content directory
mkdir -p ~/web_content
echo "<h1>Hello from Podman!</h1>" > ~/web_content/index.html
echo "<p>This is persistent storage</p>" >> ~/web_content/index.html

# Run with volume mount
podman run -d \
  --name custom-web \
  -p 8081:80 \
  -v ~/web_content:/usr/share/nginx/html:Z \
  nginx

# Test
curl http://localhost:8081

# Update content
echo "<h2>Updated content</h2>" >> ~/web_content/index.html
curl http://localhost:8081

# :Z enables SELinux relabeling
```

### Systemd Integration (Rootless):
```bash
# Create systemd user directory
mkdir -p ~/.config/systemd/user

# Generate systemd unit file
cd ~/.config/systemd/user
podman generate systemd --name custom-web --files --new

# This creates: container-custom-web.service
ls -l *.service

# Stop and remove container (systemd will recreate)
podman stop custom-web
podman rm custom-web

# Reload systemd user daemon
systemctl --user daemon-reload

# Enable and start service
systemctl --user enable container-custom-web.service
systemctl --user start container-custom-web.service

# Check status
systemctl --user status container-custom-web.service

# Enable user linger (allows services to run without login)
loginctl enable-linger $USER
loginctl show-user $USER | grep Linger

# Test
curl http://localhost:8081
```

### Test Persistence:
```bash
# Reboot system
sudo systemctl reboot

# After reboot:
systemctl --user status container-custom-web.service
curl http://localhost:8081
podman ps
```

### Container Images:
```bash
# View image details
podman inspect nginx

# View image history
podman history nginx

# Tag image
podman tag nginx mynginx:v1

# Save image to file
podman save -o nginx-backup.tar nginx

# Load image from file
podman load -i nginx-backup.tar

# Remove unused images
podman image prune
```

### Troubleshooting:
```bash
# View container logs
podman logs custom-web
podman logs --tail 50 custom-web

# Inspect container
podman inspect custom-web

# View resource usage
podman stats
podman stats custom-web

# View port mappings
podman port custom-web

# Check SELinux labels
ls -Z ~/web_content/
```

### Verification:
```bash
# List everything
podman images
podman ps -a
podman pod list
systemctl --user list-units --type=service | grep container
curl http://localhost:8081
loginctl show-user $USER | grep Linger
```
</details>

---

## 🎯 SCENARIO 11: YUM REPOSITORY MANAGEMENT (10 minutes)

### THE SITUATION:
You need to configure package repositories: enable/disable default repos, add custom repos, and manage software installation.

### YOUR TASKS:
1. List enabled repositories
2. Enable optional repositories
3. Add a custom repository
4. Install and remove packages
5. Create a local repository

<details>
<summary><b>📝 SOLUTION - Click to Reveal</b></summary>

### View Repositories:
```bash
# List enabled repositories
sudo yum repolist
sudo yum repolist enabled

# List all repositories (including disabled)
sudo yum repolist all

# Detailed view
sudo yum repolist -v

# Show repository configuration
sudo yum-config-manager --dump
```

### Enable/Disable Repositories:
```bash
# Install yum-utils
sudo yum install yum-utils -y

# Enable repository
sudo yum-config-manager --enable <repo-name>

# Disable repository
sudo yum-config-manager --disable <repo-name>

# Enable EPEL (Extra Packages for Enterprise Linux)
sudo yum install epel-release -y
sudo yum repolist | grep epel
```

### Add Custom Repository:
```bash
# Method 1: Using yum-config-manager
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Method 2: Manual creation
sudo vi /etc/yum.repos.d/custom.repo
```

Add:
```ini
[custom-repo]
name=Custom Repository
baseurl=http://192.168.100.51/repo/
enabled=1
gpgcheck=1
gpgkey=http://192.168.100.51/repo/RPM-GPG-KEY
```

```bash
# Clean and update cache
sudo yum clean all
sudo yum makecache

# Verify
sudo yum repolist
```

### Package Management:
```bash
# Search for package
sudo yum search nginx
sudo yum search web server

# Get package information
sudo yum info nginx

# Install package
sudo yum install nginx -y

# Install specific version
sudo yum install nginx-1.20.1

# Reinstall package
sudo yum reinstall nginx

# Remove package
sudo yum remove nginx

# Remove with dependencies
sudo yum autoremove nginx

# Update package
sudo yum update nginx

# Update all packages
sudo yum update -y
```

### Install from RPM File:
```bash
# Download RPM
wget https://example.com/package.rpm

# Install RPM with dependencies
sudo yum install ./package.rpm

# Or using rpm directly (no dependency resolution)
sudo rpm -ivh package.rpm
```

### Package Groups:
```bash
# List groups
sudo yum grouplist

# Install group
sudo yum groupinstall "Development Tools" -y

# Remove group
sudo yum groupremove "Development Tools"
```

### Create Local Repository:
```bash
# Install required tools
sudo yum install createrepo -y

# Create repo directory
sudo mkdir -p /var/www/html/localrepo

# Copy RPM files
sudo cp *.rpm /var/www/html/localrepo/

# Create repository metadata
sudo createrepo /var/www/html/localrepo/

# Update metadata when adding packages
sudo createrepo --update /var/www/html/localrepo/

# Configure client to use local repo
sudo vi /etc/yum.repos.d/local.repo
```

Add:
```ini
[localrepo]
name=Local Repository
baseurl=file:///var/www/html/localrepo/
enabled=1
gpgcheck=0
```

### Verification:
```bash
sudo yum repolist
sudo yum clean all
sudo yum makecache
sudo yum list available | head
```
</details>

---

## 💪 MASTERY INDICATORS

You've mastered Drill A when you can:
1. Complete all tasks in < 90 minutes
2. Have 100% configurations persist after reboot
3. Troubleshoot any issue without looking at solutions
4. Explain every command you use
5. Make NO mistakes in /etc/fstab (the #1 exam killer!)
