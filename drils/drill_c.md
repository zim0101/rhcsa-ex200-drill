# RHCSA DRILL C - Advanced Storage & Network
**Time Target: 55 minutes | Days 3, 6, 9, 12, 15, 18, 21, 24, 27, 30**

---

## 🎯 SCENARIO 1: RAID STORAGE CONFIGURATION (20 minutes)

### THE SITUATION:
Your organization requires fault-tolerant storage for critical applications. Create a RAID1 (mirrored) array for redundancy and test failure scenarios.

**Available Disks:** `/dev/vdd`, `/dev/vde`

### YOUR TASKS:
1. Install mdadm tools
2. Create RAID1 array using `/dev/vdd` and `/dev/vde` as `/dev/md0`
3. Format with ext4 filesystem
4. Mount on `/raid` and make persistent
5. Save RAID configuration
6. Simulate disk failure and verify redundancy

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Install RAID Tools:
```bash
sudo yum install mdadm -y
```

### Create RAID1 Array:
```bash
# Create RAID1 mirror
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdd /dev/vde

# Confirm with 'y'

# Watch creation progress
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
```

Add:
```
UUID=xxxx-xxxx  /raid  ext4  defaults  0 0
```

```bash
# Save RAID configuration
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf

# Test mount
sudo umount /raid
sudo mount -a
df -h | grep raid
```

### Test RAID Functionality:
```bash
# Create test data
echo "Critical data" | sudo tee /raid/important.txt
cat /raid/important.txt

# Check RAID status
sudo mdadm --detail /dev/md0
cat /proc/mdstat

# Simulate disk failure
sudo mdadm --manage /dev/md0 --fail /dev/vde

# Check degraded status
sudo mdadm --detail /dev/md0
cat /proc/mdstat

# Data still accessible!
cat /raid/important.txt

# Remove failed disk
sudo mdadm --manage /dev/md0 --remove /dev/vde

# Re-add disk (simulate replacement)
sudo mdadm --manage /dev/md0 --add /dev/vde

# Watch rebuild
watch cat /proc/mdstat    # Ctrl+C to exit

# Verify healthy status
sudo mdadm --detail /dev/md0
```

### Understanding RAID Levels:
```bash
# RAID 0: Striping - performance, no redundancy
# mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/vdd /dev/vde

# RAID 1: Mirroring - redundancy (what we did)
# mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdd /dev/vde

# RAID 5: Striping with parity - needs 3+ disks
# mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/vdd /dev/vde /dev/vdf

# RAID 6: Double parity - needs 4+ disks

# RAID 10: Mirrored stripe - needs 4+ disks
```

### Verification:
```bash
lsblk
cat /proc/mdstat
sudo mdadm --detail /dev/md0
df -h /raid
cat /raid/important.txt
```
</details>

---

## 🎯 SCENARIO 2: BOOTLOADER & BOOT TARGETS (15 minutes)

### THE SITUATION:
Modify GRUB bootloader settings and practice booting into different systemd targets for various scenarios (rescue, emergency, multi-user, graphical).

### YOUR TASKS:
1. Increase GRUB timeout to 10 seconds
2. Remove "rhgb quiet" to see boot messages
3. Add "net.ifnames=0" kernel parameter
4. Regenerate GRUB configuration
5. **Boot into rescue.target**
6. **Boot into emergency.target**
7. **Boot into multi-user.target**
8. Set graphical.target as default

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### Backup Configuration:
```bash
sudo cp /etc/default/grub /etc/default/grub.backup
```

### Edit GRUB Settings:
```bash
sudo vi /etc/default/grub
```

Change from:
```
GRUB_TIMEOUT=5
GRUB_DEFAULT=saved
GRUB_CMDLINE_LINUX="... rhgb quiet"
```

To:
```
GRUB_TIMEOUT=10
GRUB_DEFAULT=0
GRUB_CMDLINE_LINUX="... net.ifnames=0"
```

Key changes:
- `GRUB_TIMEOUT=10` - 10 second timeout
- `GRUB_DEFAULT=0` - First entry default
- Removed `rhgb quiet` - See boot messages
- Added `net.ifnames=0` - Traditional network naming

### Regenerate GRUB:
```bash
# For BIOS systems
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# For EFI systems
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# Set default entry
sudo grub2-set-default 0

# Verify
sudo grub2-editenv list
```

### Understanding Systemd Targets:

```bash
# View all targets
systemctl list-units --type=target

# View default target
systemctl get-default

# Common targets:
# - poweroff.target: Shutdown
# - rescue.target: Single-user mode with minimal services
# - emergency.target: Emergency shell with root filesystem mounted read-only
# - multi-user.target: Multi-user text mode
# - graphical.target: Multi-user with GUI
# - reboot.target: Reboot system
```

### Boot into Rescue Target:

**Method 1: Temporary (at GRUB)**
```bash
# At GRUB menu:
# 1. Press 'e' to edit
# 2. Find line starting with 'linux'
# 3. Add at end: systemd.unit=rescue.target
# 4. Press Ctrl+X to boot

# You'll be prompted for root password
# Rescue mode provides:
# - Basic system services
# - Root filesystem mounted
# - Single-user environment
```

**Method 2: From Running System**
```bash
# Switch to rescue target
sudo systemctl isolate rescue.target
# System will prompt for root password

# Return to previous target
sudo systemctl isolate multi-user.target
# or
sudo systemctl isolate graphical.target
```

### Boot into Emergency Target:

**Method 1: Temporary (at GRUB)**
```bash
# At GRUB menu:
# 1. Press 'e' to edit
# 2. Find line starting with 'linux'
# 3. Add at end: systemd.unit=emergency.target
# 4. Press Ctrl+X to boot

# Emergency mode provides:
# - Minimal environment
# - Root filesystem mounted READ-ONLY initially
# - No services running
# - Must remount rw: mount -o remount,rw /
```

**Method 2: From Running System**
```bash
# Switch to emergency target
sudo systemctl isolate emergency.target
# System will drop to emergency shell

# Exit emergency mode
exit
# System returns to default target
```

### Set and Test Default Targets:

#### Set Multi-User Target (Text Mode):
```bash
# Set as default
sudo systemctl set-default multi-user.target

# Verify
systemctl get-default

# Reboot to test
sudo systemctl reboot

# After reboot, you'll be in text mode
# To temporarily switch to graphical:
sudo systemctl isolate graphical.target
```

#### Set Graphical Target (GUI Mode):
```bash
# Set as default
sudo systemctl set-default graphical.target

# Verify
systemctl get-default

# Reboot to test
sudo systemctl reboot

# After reboot, you'll have GUI login
```

### Temporary Target Changes:
```bash
# Switch to multi-user without reboot
sudo systemctl isolate multi-user.target

# Switch to graphical without reboot
sudo systemctl isolate graphical.target

# View active target
systemctl list-units --type=target --state=active
```

### Practice All Boot Targets:

```bash
# 1. Test rescue mode
sudo systemctl isolate rescue.target
# Enter root password, explore, then exit or:
sudo systemctl isolate multi-user.target

# 2. Test emergency mode (be careful!)
sudo systemctl isolate emergency.target
# Type 'exit' to return

# 3. Switch between multi-user and graphical
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target

# 4. Check current target
systemctl list-units --type=target --state=active
```

### Target Dependencies:
```bash
# View target dependencies
systemctl list-dependencies graphical.target
systemctl list-dependencies multi-user.target
systemctl list-dependencies rescue.target

# Note: graphical.target requires multi-user.target
# multi-user.target requires basic.target
```

### Troubleshooting Boot Issues:

```bash
# If system won't boot:
# 1. At GRUB, add: systemd.unit=rescue.target
# 2. Login as root
# 3. Check logs: journalctl -xb
# 4. Fix issues
# 5. Reboot: systemctl reboot

# If graphical mode fails:
# 1. Boot to multi-user: systemd.unit=multi-user.target
# 2. Check display manager: systemctl status gdm
# 3. Fix issues
# 4. Switch to graphical: systemctl isolate graphical.target
```

### Verification After Reboot:
```bash
sudo systemctl reboot

# After reboot:
# 1. Check kernel parameters
cat /proc/cmdline

# 2. Verify boot messages visible (no rhgb quiet)
dmesg | head -50

# 3. Check default target
systemctl get-default

# 4. Test target switching
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target
```

### Quick Reference:

```bash
# Boot targets (least to most features):
# emergency.target -> rescue.target -> multi-user.target -> graphical.target

# Set default:
sudo systemctl set-default TARGET

# Switch temporarily:
sudo systemctl isolate TARGET

# At GRUB (temporary):
# Add: systemd.unit=TARGET
```
</details>

---

## 🎯 SCENARIO 3: NFS & AUTOFS CONFIGURATION (20 minutes)

### THE SITUATION:
Set up centralized file storage using NFS server and configure AutoFS for automatic on-demand mounting on clients.

**Setup:**
- Server: 192.168.100.51 (exports `/nfs/shared`)
- Client: Your system (mounts via NFS and AutoFS)

### YOUR TASKS:

**SERVER SIDE:**
1. Install NFS server
2. Create and export `/nfs/shared` directory
3. Configure firewall for NFS
4. Verify exports

**CLIENT SIDE:**
1. Install NFS client
2. Mount NFS share on `/mnt/nfs` (persistent)
3. Configure AutoFS for `/shares/data`
4. Test both mounts

<details>
<summary><b>🔍 SOLUTION - Click to Reveal</b></summary>

### NFS SERVER CONFIGURATION:

#### Install and Configure:
```bash
# Install NFS server
sudo yum install nfs-utils -y

# Create shared directory
sudo mkdir -p /nfs/shared
sudo chmod 755 /nfs/shared

# Create test file
echo "NFS test data" | sudo tee /nfs/shared/testfile.txt
```

#### Configure Exports:
```bash
sudo vi /etc/exports
```

Add:
```
/nfs/shared  192.168.100.0/24(rw,sync,no_root_squash,no_subtree_check)
```

Options explained:
- `rw` - Read/write access
- `sync` - Synchronous writes
- `no_root_squash` - Root on client = root on server
- `no_subtree_check` - Better performance

```bash
# Apply exports
sudo exportfs -avr

# Verify
sudo exportfs -v
cat /var/lib/nfs/etab
```

#### Start NFS Services:
```bash
sudo systemctl enable --now nfs-server
sudo systemctl enable --now rpcbind
sudo systemctl status nfs-server
```

#### Configure Firewall:
```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-services
```

### NFS CLIENT CONFIGURATION:

#### Install Client:
```bash
sudo yum install nfs-utils -y

# Check server exports
showmount -e 192.168.100.51
```

#### Method 1: Traditional NFS Mount:
```bash
# Create mount point
sudo mkdir -p /mnt/nfs

# Mount temporarily
sudo mount -t nfs 192.168.100.51:/nfs/shared /mnt/nfs

# Verify
df -h | grep nfs
ls -l /mnt/nfs
cat /mnt/nfs/testfile.txt

# Test write
echo "Client test" | sudo tee /mnt/nfs/clientfile.txt
```

#### Make NFS Mount Persistent:
```bash
sudo vi /etc/fstab
```

Add:
```
192.168.100.51:/nfs/shared  /mnt/nfs  nfs  defaults,_netdev  0 0
```

Note: `_netdev` waits for network before mounting

```bash
# Test without reboot
sudo umount /mnt/nfs
sudo mount -a
df -h | grep nfs
```

#### Method 2: AutoFS Configuration:

Install AutoFS:
```bash
sudo yum install autofs -y
```

Configure Master Map:
```bash
sudo vi /etc/auto.master
```

Add:
```
/shares  /etc/auto.nfs  --timeout=60
```

Create NFS Map:
```bash
sudo vi /etc/auto.nfs
```

Add:
```
data  -rw,soft,intr,rsize=8192,wsize=8192  192.168.100.51:/nfs/shared
```

Format: `key  options  location`

Start AutoFS:
```bash
sudo systemctl enable --now autofs
sudo systemctl status autofs
```

#### Test AutoFS:
```bash
# Directory doesn't exist yet
ls /shares    # Empty or doesn't exist

# Access triggers automatic mount
ls /shares/data
cat /shares/data/testfile.txt

# Verify mounted
df -h | grep data
mount | grep data

# After 60 seconds of inactivity, auto-unmounts
sleep 65
df -h | grep data    # Should be gone

# Access again - remounts automatically
cat /shares/data/testfile.txt
```

#### Advanced AutoFS - Multiple Shares:
```bash
sudo vi /etc/auto.nfs
```

Add multiple entries:
```
data      -rw,soft  192.168.100.51:/nfs/shared
backup    -ro,soft  192.168.100.51:/nfs/backup
docs      -rw,soft  192.168.100.51:/nfs/documents
```

```bash
sudo systemctl reload autofs

# Test all mounts
ls /shares/data
ls /shares/backup
ls /shares/docs
```

#### Wildcard AutoFS (Advanced):
```bash
# In /etc/auto.master:
/nfs  /etc/auto.nfs

# In /etc/auto.nfs:
*  -rw,soft  192.168.100.51:/nfs/&
```

This mounts `/nfs/anything` as `192.168.100.51:/nfs/anything`

### Troubleshooting:

#### Server Side:
```bash
sudo systemctl status nfs-server
sudo exportfs -v
sudo journalctl -u nfs-server
rpcinfo -p
sudo tail -f /var/log/messages
```

#### Client Side:
```bash
sudo systemctl status autofs
showmount -e 192.168.100.51
sudo journalctl -u autofs
mount | grep nfs

# Test manual mount
sudo mount -t nfs -v 192.168.100.51:/nfs/shared /mnt/test

# Debug AutoFS
sudo automount -f -v    # Run in foreground
```

### Verification:

#### Server:
```bash
sudo exportfs -v
systemctl status nfs-server
showmount -e localhost
```

#### Client:
```bash
# Traditional mount
df -h | grep /mnt/nfs
cat /mnt/nfs/testfile.txt

# AutoFS
ls /shares/data
cat /shares/data/testfile.txt
systemctl status autofs

# Both should work after reboot
sudo systemctl reboot
# After reboot, verify both mounts
```
</details>

---

## ✅ DRILL C COMPLETION CHECKLIST

- [ ] RAID1 array created and working
- [ ] RAID configuration saved to /etc/mdadm.conf
- [ ] RAID mount persistent in /etc/fstab
- [ ] Disk failure test successful
- [ ] GRUB timeout increased
- [ ] Boot messages visible (rhgb quiet removed)
- [ ] Kernel parameters added
- [ ] GRUB regenerated successfully
- [ ] Rescue target tested
- [ ] Emergency target tested
- [ ] Multi-user target working
- [ ] Default target set correctly
- [ ] NFS server installed and configured
- [ ] NFS exports working
- [ ] NFS firewall rules applied
- [ ] Traditional NFS mount persistent
- [ ] AutoFS installed and configured
- [ ] AutoFS auto-mounting working
- [ ] System rebooted - all persists

**Time Completed: _____ minutes**

---

## 💪 MASTERY INDICATORS

✅ Complete in < 55 minutes
✅ RAID array survives disk failure
✅ Can boot into all systemd targets
✅ NFS mounts work after reboot
✅ AutoFS mounts on-demand
✅ GRUB changes applied correctly
✅ All configurations persistent