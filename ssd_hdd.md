### Copy the SSD_99-mint-hardeing.conf to /etc/sysctl.d/99-mint-hardening.conf


# 1. Automatically find your mechanical SDD & create the rule
# Replace nvme0n1 or sda with your actual SSD device
#DRIVE="nvme0n1"  # "nvme0n1" for nvm drive
DRIVE="sda"  # or "sda" for SATA SSD
echo "ACTION==\"add|change\", KERNEL==\"${DRIVE}\", ATTR{queue/scheduler}=\"none\"" | sudo tee /etc/udev/rules.d/60-ssd-scheduler.rules
# 💡 none = optimal for NVMe & modern SATA SSDs. Falls back to mq-deadline automatically if unsupported.

# Enable Weekly TRIM (Extends SSD Lifespan)
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer

# 1. Verify scheduler 
cat /sys/block/$(lsblk -d -o NAME | grep -E "nvme|sda" | head -1)/queue/scheduler
# Should return: [none] mq-deadline 

# 2. Check TRIM support & status
sudo fstrim -v /
#Should return: /: 317 GiB (340357988352 bytes) trimmed


# 3. Monitor SSD wear (requires smartmontools)
sudo apt install smartmontools
sudo smartctl -a /dev/sda | grep -E "Percentage|Media_Wearout"

# TRIM Strategy (Critical for SSD Longevity)
# Enable weekly TRIM (runs automatically, low overhead)
sudo systemctl enable --now fstrim.timer

# Check timer status
systemctl status fstrim.timer

# Run manual TRIM to verify support & see savings
sudo fstrim -v /
# ✅ Output example: "/: 12.5 GiB (13428453376 bytes) trimmed"

# Rollback if needed
#sudo rm /etc/udev/rules.d/60-io-scheduler.rules
#echo mq-deadline | sudo tee /sys/block/${DRIVE}/queue/scheduler

# Safe /etc/fstab Optimization Template
# Backup fstab
sudo cp /etc/fstab /etc/fstab.bak-$(date +%F)

sudo nano /etc/fstab

# Find your ext4 partitions (usually /). Modify the 4th column (mount options):
# 📜 BEFORE (Example)
# /dev/mapper/vgmint-root /               ext4    errors=remount-ro 0       1
# or
# UUID = ################# /              ext4    errors=remount-ro 0       1   

# ✅ AFTER (Optimized)
# /dev/mapper/vgmint-root /               ext4    defaults,noatime,errors=remount-ro 0       1
# or
# UUID = ################# /              ext4    defaults,noatime,errors=remount-ro 0       1              

# Verify Syntax Before Rebooting
sudo findmnt --verify --tab-file /etc/fstab
# ✅ Should output: "SUCCESS"

# If it says ERROR: Restore immediately:
sudo cp /etc/fstab.bak-$(date +%F) /etc/fstab

# Reload systemd's fstab cache
sudo systemctl daemon-reload


# Backup GRUB Config
sudo cp /etc/default/grub /etc/default/grub.bak

# Edit GRUB
sudo nano /etc/default/grub

# Find this line:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

# FOR SDA - Change it to:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash zswap.enabled=1 zswap.compressor=zstd"

# ~3:1 compression ratio with minimal CPU overhead. Writes fewer pages to SSD than lz4 or lzo


# ⚠️ Only add nvme_core.default_ps_max_latency_us=0 if you experience random 1-3s freezes during light disk access.
# It keeps the drive in its highest performance state at the cost of slightly higher idle power draw.
# If your SSD is NVMe (/dev/nvme0n1), add this to prevent power-state transition latency:

#GRUB_CMDLINE_LINUX_DEFAULT="quiet splash zswap.enabled=1 zswap.compressor=zstd nvme_core.default_ps_max_latency_us=0"

sudo update-grub
sudo reboot


# After logging back in, verify the options are active:
mount | grep " / "
# ✅ Should show something similar: dev/sda5 on / type ext4 (rw,noatime,errors=remount-ro)

# Quick SSD Health Checklist
# 1. Confirm I/O scheduler is 'none'
cat /sys/block/sda/queue/scheduler
# [nvme] or sata SSD should show [none] mq-deadline 

# 2. Check TRIM support & alignment
sudo fstrim -v /

# 3. Monitor SSD wear (requires smartmontools)
sudo smartctl -a /dev/sda | grep -iE "percentage|wearout|total_lba"

# 4. Verify GRUB applied correctly
cat /proc/cmdline | grep -o "zswap.enabled=1 zswap.compressor=zstd"

# 5. Verify TRIM is active
systemctl status fstrim.timer | grep "active"
