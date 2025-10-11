# RHCSA DRILL A - Core System Administration
**Time Target: 180 minutes | Days 1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21, 23, 25, 27, 29**

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
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

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
systemctl get-default                          # Check current

sudo systemctl set-default multi-user.target  # Set to text mode

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

## 🎯 SCENARIO 2: YUM REPOSITORY MANAGEMENT (10 minutes)

### THE SITUATION:
You need to configure package repositories for software installation. The server needs access to both official RHEL repositories and EPEL for additional packages.

### YOUR TASKS:
1. List all enabled repositories
2. Add a custom local repository at `file:///repo/local`
3. Install a test package from the custom repository
4. Update repository cache

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### View Current Repositories:
```bash
# List enabled repositories
sudo yum repolist
sudo yum repolist enabled

# List all repositories (including disabled)
sudo yum repolist all

# Detailed view
sudo yum repolist -v
```

### Add Custom Repository:
```bash
# Create repository file
sudo vi /etc/yum.repos.d/local.repo
```

Add:
```ini
[local-repo]
name=Local Repository
baseurl=file:///repo/local
enabled=1
gpgcheck=0
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
sudo yum search httpd

# Get package information
sudo yum info httpd

# Install package
sudo yum install httpd -y

# Update all packages
sudo yum update -y
```

### Verification:
```bash
sudo yum repolist
sudo yum list installed | grep httpd
```
</details>

---

## 🎯 SCENARIO 3: USER AND GROUP MANAGEMENT WITH SUDO (20 minutes)

### THE SITUATION:
Your organization needs proper user management with different access levels. Create users, configure password policies, and set up sudo access for system administration tasks.

### YOUR TASKS:
1. Create users: `john`, `mary`, `developer1`
2. Create groups: `developers`, `admins`
3. Configure password aging for `john`: max 90 days, min 7 days, warn 14 days
4. Force `mary` to change password on next login
5. Add `john` to wheel group for sudo access
6. Create custom sudo rule for `developer1` to restart httpd only
7. Test sudo access for all users

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Create Users and Groups:
```bash
# Create users
sudo useradd john
sudo useradd mary
sudo useradd developer1

# Set passwords
echo "Password123!" | sudo passwd --stdin john
echo "Password123!" | sudo passwd --stdin mary
echo "Password123!" | sudo passwd --stdin developer1

# Create groups
sudo groupadd developers
sudo groupadd admins

# Add users to groups
sudo usermod -aG developers john
sudo usermod -aG developers developer1
sudo usermod -aG admins mary

# Verify
groups john
groups mary
```

### Configure Password Aging:
```bash
# Set password aging for john
sudo chage -M 90 john      # Max 90 days
sudo chage -m 7 john       # Min 7 days
sudo chage -W 14 john      # Warn 14 days
sudo chage -I 30 john      # Inactive 30 days

# View settings
sudo chage -l john

# Force mary to change password
sudo chage -d 0 mary
```

### Configure Sudo Access:
```bash
# Add john to wheel group
sudo usermod -aG wheel john

# Create custom sudo rule for developer1
sudo visudo -f /etc/sudoers.d/developer1
```

Add:
```
developer1 ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd
```

```bash
# Set permissions
sudo chmod 0440 /etc/sudoers.d/developer1

# Verify syntax
sudo visudo -c
```

### Test Sudo Access:
```bash
# Test john (full sudo)
su - john
sudo whoami                 # Should work
exit

# Test developer1 (limited)
su - developer1
sudo systemctl status httpd # Should work
sudo ls /root              # Should FAIL
exit
```

### Verification:
```bash
sudo chage -l john
sudo -l -U john
sudo -l -U developer1
```
</details>

---

## 🎯 SCENARIO 4: NETWORK CONFIGURATION (20 minutes)

### THE SITUATION:
Your server needs static IP addressing for production. Configure IPv4, IPv6, hostname, and DNS settings.

**Network Details:**
- IPv4: 192.168.100.50/24
- Gateway: 192.168.100.1
- DNS: 8.8.8.8, 8.8.4.4
- IPv6: 2001:db8::50/64
- Gateway: 2001:db8::1
- Hostname: prod-server.example.com

### YOUR TASKS:
1. Configure static IPv4 and IPv6 addresses
2. Set hostname to `prod-server.example.com`
3. Configure DNS servers
4. Add hostname to `/etc/hosts`
5. Verify network connectivity

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Configure Network with nmcli:
```bash
# List connections
nmcli con show

# Set connection name
CONN_NAME="System eth0"  # Replace with your connection

# Configure IPv4
sudo nmcli con mod "$CONN_NAME" ipv4.addresses 192.168.100.50/24
sudo nmcli con mod "$CONN_NAME" ipv4.gateway 192.168.100.1
sudo nmcli con mod "$CONN_NAME" ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli con mod "$CONN_NAME" ipv4.method manual

# Configure IPv6
sudo nmcli con mod "$CONN_NAME" ipv6.addresses 2001:db8::50/64
sudo nmcli con mod "$CONN_NAME" ipv6.gateway 2001:db8::1
sudo nmcli con mod "$CONN_NAME" ipv6.method manual

# Apply changes
sudo nmcli con up "$CONN_NAME"
```

### Set Hostname:
```bash
# Set hostname
sudo hostnamectl set-hostname prod-server.example.com

# Verify
hostnamectl status
hostname -f
```

### Configure /etc/hosts:
```bash
sudo vi /etc/hosts
```

Add:
```
192.168.100.50    prod-server.example.com prod-server
2001:db8::50      prod-server.example.com prod-server
```

### Verification:
```bash
ip addr show
ip route show
ip -6 route show
cat /etc/resolv.conf
hostnamectl
ping -c 3 8.8.8.8
ping -c 3 google.com
```
</details>

---

## 🎯 SCENARIO 5: DISK PARTITIONING FOR NEW SERVER (20 minutes)

### THE SITUATION:
You're setting up a new server with additional storage. You need to create partitions using `cfdisk` for both MBR and GPT layouts.

**Available Disks:**
- `/dev/vdb` (10GB) - Use MBR
- `/dev/vdc` (10GB) - Use GPT

### YOUR TASKS:
1. On `/dev/vdb` (MBR): Create 4GB, 4GB swap, and 2GB LVM partitions
2. On `/dev/vdc` (GPT): Create two 5GB partitions, mark second as LVM
3. Verify all partitions are recognized

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Create MBR Partitions with cfdisk:
```bash
# Start cfdisk
sudo cfdisk /dev/vdb

# Select: dos (MBR)
# Create partitions using cfdisk interface:
# - New -> 4G -> Type: Linux
# - New -> 4G -> Type: Linux swap
# - New -> 2G -> Type: Linux LVM

# Write changes and quit

# Inform kernel
sudo partprobe /dev/vdb

# Verify
lsblk /dev/vdb
```

### Create GPT Partitions with cfdisk:
```bash
# Start cfdisk
sudo cfdisk /dev/vdc

# Select: gpt
# Create partitions:
# - New -> 5G -> Type: Linux filesystem
# - New -> 5G -> Type: Linux LVM

# Write and quit

# Verify
sudo partprobe /dev/vdc
lsblk /dev/vdc
```

### Final Verification:
```bash
lsblk
sudo fdisk -l /dev/vdb
sudo fdisk -l /dev/vdc
```
</details>

---

## 🎯 SCENARIO 6: ENTERPRISE STORAGE WITH LVM (25 minutes)

### THE SITUATION:
Configure LVM storage with various extension and reduction scenarios for both XFS and ext4 filesystems.

### YOUR TASKS:
1. Create PVs from `/dev/vdb3` and `/dev/vdc2`
2. Create VG `datavg` using both PVs
3. Create LVs:
   - `datalv`: 3GB with XFS
   - `backuplv`: 2GB with ext4
4. Mount on `/data` and `/backup`, make persistent
5. **Extend `datalv` to 100% of VG**
6. **Extend `backuplv` by +1GB**
7. **Reduce `backuplv` by 1GB** (ext4 supports this)

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Create Physical Volumes:
```bash
sudo pvcreate /dev/vdb3 /dev/vdc2

# Verify
sudo pvs
sudo pvdisplay
```

### Create Volume Group:
```bash
sudo vgcreate datavg /dev/vdb3 /dev/vdc2

# Verify
sudo vgs
sudo vgdisplay datavg
```

### Create Logical Volumes:
```bash
# Create datalv (3GB, XFS)
sudo lvcreate -L 3G -n datalv datavg
sudo mkfs.xfs /dev/datavg/datalv

# Create backuplv (2GB, ext4)
sudo lvcreate -L 2G -n backuplv datavg
sudo mkfs.ext4 /dev/datavg/backuplv

# Verify
sudo lvs
lsblk -f | grep datavg
```

### Mount and Make Persistent:
```bash
# Create mount points
sudo mkdir -p /data /backup

# Mount
sudo mount /dev/datavg/datalv /data
sudo mount /dev/datavg/backuplv /backup

# Get UUIDs
sudo blkid /dev/datavg/datalv
sudo blkid /dev/datavg/backuplv

# Edit fstab
sudo vi /etc/fstab
```

Add:
```
UUID=xxxx-xxxx  /data    xfs   defaults  0 0
UUID=yyyy-yyyy  /backup  ext4  defaults  0 0
```

```bash
# Test
sudo umount /data /backup
sudo mount -a
df -h | grep -E 'data|backup'
```

### Extend datalv to 100% of VG:
```bash
# Check available space
sudo vgs datavg

# Extend to 100% of VG
sudo lvextend -l +100%FREE /dev/datavg/datalv

# Grow XFS filesystem
sudo xfs_growfs /data

# Verify
df -h /data
sudo lvs
```

### Extend backuplv by +1GB:
```bash
# Extend LV
sudo lvextend -L +1G /dev/datavg/backuplv

# Resize ext4 filesystem
sudo resize2fs /dev/datavg/backuplv

# Verify
df -h /backup
sudo lvs
```

### Reduce backuplv by 1GB:
```bash
# IMPORTANT: Only ext4 supports shrinking, NOT xfs!
# Unmount first
sudo umount /backup

# Check filesystem
sudo e2fsck -f /dev/datavg/backuplv

# Resize filesystem first
sudo resize2fs /dev/datavg/backuplv 2G

# Then reduce LV
sudo lvreduce -L 2G /dev/datavg/backuplv
# Confirm with 'y'

# Remount
sudo mount /backup

# Verify
df -h /backup
sudo lvs
```

### Verification:
```bash
sudo pvs
sudo vgs
sudo lvs
df -h
lsblk
```
</details>

---

## 🎯 SCENARIO 7: SWAP SPACE CONFIGURATION (10 minutes)

### THE SITUATION:
The server needs additional swap space for memory-intensive operations.

### YOUR TASKS:
1. Configure `/dev/vdb2` as swap partition
2. Create a 2GB swap file at `/swapfile`
3. Make both persistent
4. Verify total swap is available

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Configure Swap Partition:
```bash
# Format as swap
sudo mkswap /dev/vdb2

# Enable
sudo swapon /dev/vdb2

# Verify
swapon --show
free -h
```

### Create Swap File:
```bash
# Create 2GB file
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress

# Secure permissions
sudo chmod 600 /swapfile

# Format as swap
sudo mkswap /swapfile

# Enable
sudo swapon /swapfile

# Verify
swapon --show
free -h
```

### Make Persistent:
```bash
sudo vi /etc/fstab
```

Add:
```
/dev/vdb2     none    swap    defaults    0 0
/swapfile     none    swap    defaults    0 0
```

```bash
# Test
sudo swapoff -a
sudo swapon -a
swapon --show
```

### Verification:
```bash
swapon --show
free -h
cat /proc/swaps
```
</details>

---

## 🎯 SCENARIO 8: VFAT FILESYSTEM & USB STORAGE (10 minutes)

### THE SITUATION:
Configure VFAT filesystem for Windows compatibility and USB drive usage.

### YOUR TASKS:
1. Create VFAT filesystem on `/dev/vdd1`
2. Mount on `/mnt/usb`
3. Test read/write access
4. Make mount persistent with proper options
5. Verify after reboot

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Create VFAT Filesystem:
```bash
# Create partition if needed (use cfdisk)
sudo cfdisk /dev/vdd
# Create partition, write, quit

sudo partprobe /dev/vdd

# Format as VFAT
sudo mkfs.vfat /dev/vdd1

# Verify
sudo blkid /dev/vdd1
```

### Mount and Test:
```bash
# Create mount point
sudo mkdir -p /mnt/usb

# Mount
sudo mount /dev/vdd1 /mnt/usb

# Test write
echo "Test file" | sudo tee /mnt/usb/test.txt
cat /mnt/usb/test.txt
```

### Make Persistent:
```bash
# Get UUID
sudo blkid /dev/vdd1

sudo vi /etc/fstab
```

Add:
```
UUID=xxxx-xxxx  /mnt/usb  vfat  defaults,umask=0022,uid=1000,gid=1000  0 0
```

```bash
# Test
sudo umount /mnt/usb
sudo mount -a
df -h | grep usb
```

### Verification:
```bash
df -h | grep usb
cat /mnt/usb/test.txt
ls -l /mnt/usb
```
</details>

---

## 🎯 SCENARIO 9: FIREWALL CONFIGURATION (15 minutes)

### THE SITUATION:
Configure firewalld for web server with custom SSH port and trusted internal network.

### YOUR TASKS:
1. Allow HTTP (80), HTTPS (443)
2. Allow custom SSH port 2222
3. Allow application port 8080
4. Create trusted zone for 10.0.0.0/24
5. Make all rules persistent

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Enable Firewalld:
```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --state
```

### Allow Services and Ports:
```bash
# Add HTTP and HTTPS
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Remove default SSH, add custom port
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --permanent --add-port=2222/tcp

# Add application port
sudo firewall-cmd --permanent --add-port=8080/tcp

# Reload
sudo firewall-cmd --reload
```

### Configure Trusted Zone:
```bash
# Add internal network to trusted zone
sudo firewall-cmd --permanent --zone=trusted --add-source=10.0.0.0/24

# Reload
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=trusted --list-all
```

### Verification:
```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --list-all-zones
```
</details>

---

## 🎯 SCENARIO 10: SSH SECURITY HARDENING (15 minutes)

### THE SITUATION:
Harden SSH: disable root login, move to port 2222, require key-based authentication.

### YOUR TASKS:
1. Generate SSH key pair (RSA, 4096 bits)
2. Configure SSH port 2222
3. Disable root login and password authentication
4. Update SELinux for port 2222
5. Test secure connection

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Generate SSH Key:
```bash
ssh-keygen -t rsa -b 4096 -C "admin@prod-server.example.com"

# Set permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

### Configure SSH Server:
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

sudo vi /etc/ssh/sshd_config
```

Modify:
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

### Update SELinux:
```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo semanage port -l | grep ssh
```

### Reload SSH:
```bash
sudo sshd -t
sudo systemctl reload sshd
sudo ss -tulpn | grep 2222
```

### Verification:
```bash
ssh -p 2222 localhost
sudo systemctl status sshd
```
</details>

---

## 🎯 SCENARIO 11: SELINUX CONFIGURATION (20 minutes)

### THE SITUATION:
Configure SELinux for Apache web server with custom directory and port.

### YOUR TASKS:
1. Verify SELinux is enforcing
2. Create `/web` directory for Apache
3. Fix SELinux context for `/web`
4. Allow Apache network connections (boolean)
5. Configure SELinux port label for port 8080
6. Verify all settings

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Check SELinux Status:
```bash
getenforce
sestatus

# Set enforcing if needed
sudo setenforce 1
```

### Create Web Directory:
```bash
sudo mkdir -p /web/html
echo "<h1>Test Page</h1>" | sudo tee /web/html/index.html

# Check context (wrong)
ls -Z /web
```

### Fix SELinux Context:
```bash
# Add context rule
sudo semanage fcontext -a -t httpd_sys_content_t '/web(/.*)?'

# Apply
sudo restorecon -Rv /web

# Verify
ls -Z /web/html/
```

### Configure SELinux Booleans:
```bash
# Check current value
getsebool httpd_can_network_connect

# Enable
sudo setsebool -P httpd_can_network_connect on

# Verify
getsebool httpd_can_network_connect
```

### Configure Port Label:
```bash
# View current ports
sudo semanage port -l | grep http_port_t

# Add port 8080
sudo semanage port -a -t http_port_t -p tcp 8080

# Verify
sudo semanage port -l | grep http_port_t
```

### Verification:
```bash
getenforce
ls -Z /web/html/
getsebool httpd_can_network_connect
sudo semanage port -l | grep 8080
```
</details>

---

## 🎯 SCENARIO 12: CONTAINER MANAGEMENT (15 minutes)

### THE SITUATION:
Deploy containerized nginx with persistent storage and systemd service.

### YOUR TASKS:
1. Install Podman
2. Pull and run nginx container
3. Configure persistent volume
4. Create systemd service for rootless container
5. Test persistence across reboots

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Install Podman:
```bash
sudo yum install podman -y
podman --version
```

### Run Container with Persistent Storage:
```bash
# Create content directory
mkdir -p ~/web_content
echo "<h1>Hello from Podman!</h1>" > ~/web_content/index.html

# Pull and run
podman pull docker.io/library/nginx:latest
podman run -d --name custom-web -p 8081:80 -v ~/web_content:/usr/share/nginx/html:Z nginx

# Test
curl http://localhost:8081
```

### Create Systemd Service:
```bash
# Create directory
mkdir -p ~/.config/systemd/user

# Generate service
cd ~/.config/systemd/user
podman generate systemd --name custom-web --files --new

# Stop and remove container
podman stop custom-web
podman rm custom-web

# Enable service
systemctl --user daemon-reload
systemctl --user enable --now container-custom-web.service

# Enable linger
loginctl enable-linger $USER
```

### Verification:
```bash
systemctl --user status container-custom-web.service
curl http://localhost:8081
podman ps
```
</details>

---

## ✅ DRILL A COMPLETION CHECKLIST

- [ ] Root password reset successful
- [ ] Boot target set to multi-user
- [ ] YUM repositories configured
- [ ] Users and groups created
- [ ] Password aging configured
- [ ] Sudo access working
- [ ] Network configured with static IPs
- [ ] Hostname and /etc/hosts configured
- [ ] Partitions created with cfdisk
- [ ] LVM configured (extend 100%, +size, -size)
- [ ] Swap partition and file active
- [ ] VFAT filesystem mounted
- [ ] Firewall rules applied
- [ ] SSH hardened on port 2222
- [ ] SELinux contexts fixed
- [ ] Container running as systemd service
- [ ] System rebooted - all persists

**Time Completed: _____ minutes**

---

## 💪 MASTERY INDICATORS

✅ Complete in < 180 minutes

✅ 100% configurations persist after reboot

✅ No /etc/fstab errors

✅ All LVM operations successful

✅ SELinux remains enforcing

✅ All services start on boot