# Copy the HDD_99-mint-hardeing.conf to /etc/sysctl.d/99-mint-hardening.conf


# bfq I/O Scheduler udev Rule (HDD-Optimized)
# Reduces mechanical head-seeking thrashing during multitasking.

# 1. Automatically find your mechanical HDD & create the rule
HDD=$(lsblk -d -o NAME,ROTA | awk '$2==1 {print $1}' | head -1)
echo "ACTION==\"add|change\", KERNEL==\"${HDD}\", ATTR{queue/scheduler}=\"bfq\"" | sudo tee /etc/udev/rules.d/60-io-scheduler.rules

# 2. Apply immediately (replace sda below with the detected drive if different)
echo bfq | sudo tee /sys/block/${HDD}/queue/scheduler

# 3. Verify it's active
cat /sys/block/${HDD}/queue/scheduler
# ✅ Should show: none mq-deadline kyber [bfq]

# Rollback if needed
#sudo rm /etc/udev/rules.d/60-io-scheduler.rules
#echo mq-deadline | sudo tee /sys/block/${HDD}/queue/scheduler

# Safe /etc/fstab Optimization Template
# Cuts unnecessary metadata writes by ~30% and batches disk flushes for smoother HDD performance.
# Backup fstab
sudo cp /etc/fstab /etc/fstab.bak-$(date +%F)

sudo nano /etc/fstab

# Find your ext4 partitions (usually /). Modify the 4th column (mount options):
# 📜 BEFORE (Example)
# /dev/mapper/vgmint-root /               ext4    errors=remount-ro 0       1
# or
# UUID = ################# /              ext4    errors=remount-ro 0       1              

# ✅ AFTER (Optimized)
# /dev/mapper/vgmint-root /               ext4    defaults,noatime,commit=60,errors=remount-ro 0       1
# or
# UUID = ################# /              ext4    defaults,noatime,commit=60,errors=remount-ro 0       1         


# Verify Syntax Before Rebooting
sudo findmnt --verify --tab-file /etc/fstab
# ✅ Should output: "SUCCESS"

# If it says ERROR: Restore immediately:
# sudo cp /etc/fstab.bak-$(date +%F) /etc/fstab

# Reload systemd's fstab cache
sudo systemctl daemon-reload


# Backup GRUB Config
sudo cp /etc/default/grub /etc/default/grub.bak

# Edit GRUB
sudo nano /etc/default/grub

# Find this line:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

# Change it to:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash zswap.enabled=1 zswap.compressor=lz4"

sudo update-grub
sudo reboot


****************************************************************
# Verify After Reboot
# After logging back in, verify the options are active:
mount | grep " / "

# 1. Check kernel loaded it
dmesg | grep zswap
# ✅ Expected: "zswap: loaded using pool lz4/zsmalloc"

# 2. Confirm it's enabled
cat /sys/module/zswap/parameters/enabled
# ✅ Should output: Y

# 3. Verify sysctl persistence
sysctl -p /etc/sysctl.d/99-mint-hardening.conf 2>&1 | grep -i error

# 4. Check I/O scheduler
cat /sys/block/$(lsblk -d -o NAME,ROTA | awk '$2==1 {print $1}' | head -1)/queue/scheduler

# 5. Verify mount options
mount | grep " / " | grep -o "noatime.*commit=60"

# 6. Monitor I/O wait (should stay <5% during normal desktop use)
vmstat 1 5 | awk 'NR>2 {print $16"% wa"}'


# Never reboot after editing /etc/fstab without running findmnt --verify first.
# If dual-booting Windows, disable "Fast Startup" in Windows to prevent filesystem locks on shared drives.
# Keep your backup drive connected for Timeshift until you've tested everything for a few days.