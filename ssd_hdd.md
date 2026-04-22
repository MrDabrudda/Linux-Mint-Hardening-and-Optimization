# Linux Mint SSD Optimization Guide

## Use this configuration guide for SSD and NVMe HDD only

## 1. Apply Hardened Sysctl Configuration
Copy the pre-configured hardening file to the system directory:
```bash
sudo cp SSD_99-mint-hardening.conf /etc/sysctl.d/99-mint-hardening.conf
```
Apply immediately
```bash
sudo sysctl --system
```
Verify key parameters
```bash
sysctl vm.swappiness vm.dirty_ratio vm.vfs_cache_pressure net.ipv4.tcp_congestion_control net.core.default_qdisc
```
✅ **Expected output:** `ault_qdisc
vm.swappiness = 10
vm.dirty_ratio = 20
vm.vfs_cache_pressure = 50
net.ipv4.tcp_congestion_control = cubic
net.core.default_qdisc = fq
`
## 2. Configure `none` I/O Scheduler & TRIM (SSD-Optimized)
> Modern SSDs handle I/O optimization internally. The `none` scheduler passes requests directly to the drive with zero CPU overhead.

### Step 1: Create udev Rule for `none` Scheduler
Replace `sda` with your actual SSD device (`nvme0n1` for NVMe, `sda`/`sdb` for SATA).
```bash
DRIVE="sda"  # Change to your actual SSD device
echo "ACTION==\"add|change\", KERNEL==\"${DRIVE}\", ATTR{queue/scheduler}=\"none\"" | sudo tee /etc/udev/rules.d/60-ssd-scheduler.rules
```
💡 `none` is optimal for NVMe & modern SATA SSDs. Falls back to `mq-deadline` automatically if unsupported.

### Step 2: Enable Weekly TRIM (Extends SSD Lifespan)
```bash
sudo systemctl enable --now fstrim.timer
```

### Step 3: Verify Scheduler & TRIM Status
```bash
cat /sys/block/$(lsblk -d -o NAME | grep -E "nvme|sda" | head -1)/queue/scheduler
```
✅ **Expected output:** `none [mq-deadline]`

### 🔄 Rollback (If Needed)
```bash
sudo rm /etc/udev/rules.d/60-ssd-scheduler.rules
echo mq-deadline | sudo tee /sys/block/${DRIVE}/queue/scheduler
```

## 3. Optimize `/etc/fstab` for SSD Performance
> Removes unnecessary metadata writes and relies on scheduled TRIM instead of continuous discard.

### Backup fstab
```bash
sudo cp /etc/fstab /etc/fstab.bak-$(date +%F)
```

### Edit fstab
```bash
sudo nano /etc/fstab
```

### Modify Mount Options
Find your root partition (`ext4`) and update the **4th column** (mount options):

**📜 BEFORE:**
```text
/dev/mapper/vgmint-root /  ext4  errors=remount-ro  0  1
# or
UUID=####################  /  ext4  errors=remount-ro  0  1
```

**✅ AFTER:**
```text
/dev/mapper/vgmint-root /  ext4  defaults,noatime,errors=remount-ro  0  1
# or
UUID=####################  /  ext4  defaults,noatime,errors=remount-ro  0  1
```

### Verify Syntax Before Rebooting
```bash
sudo findmnt --verify --tab-file /etc/fstab
```
✅ **Expected:** `0 parse errors, 0 errors` (1 warning about systemd cache is normal)  
❌ **If ERROR:** Restore immediately: `sudo cp /etc/fstab.bak-$(date +%F) /etc/fstab`

### Reload Systemd Cache
```bash
sudo systemctl daemon-reload
```

## 4. Enable `zswap` via GRUB (SSD-Optimized)
Compresses swapped pages in RAM before writing to the SSD. `zstd` offers a ~3:1 compression ratio with minimal CPU overhead, writing fewer pages to flash storage than `lz4` or `lzo`.

### Backup GRUB Config
```bash
sudo cp /etc/default/grub /etc/default/grub.bak
```

### Edit GRUB
```bash
sudo nano /etc/default/grub
```

### Modify Kernel Command Line
Find this line:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

**✅ For SATA SSD (`/dev/sda`):**
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash zswap.enabled=1 zswap.compressor=zstd"
```

**⚠️ For NVMe SSD (`/dev/nvme0n1`):** Only add the `nvme_core` parameter if you experience random 1–3s freezes during light disk access. It prevents power-state transition latency at the cost of slightly higher idle power draw.
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash zswap.enabled=1 zswap.compressor=zstd nvme_core.default_ps_max_latency_us=0"
```

### Update & Reboot
```bash
sudo update-grub
sudo reboot
```

## 5. Post-Reboot Verification
Run these commands after logging back in:

### 1. Verify Mount Options
```bash
mount | grep " / "
```
✅ **Expected:** `...../ type ext4 (rw,noatime,errors=remount-ro)
`

### 2. Confirm I/O Scheduler
```bash
cat /sys/block/sda/queue/scheduler
```
✅ **Expected:** `[none] mq-deadline`

### 3. Check TRIM Support & Status
```bash
sudo fstrim -v /
```
✅ **Output example:** `/: 12.5 GiB (13428453376 bytes) trimmed`

```bash
systemctl status fstrim.timer | grep "active"
```
✅ **Should show:** `active (running)`

### 4. Verify GRUB Applied Correctly
```bash
cat /proc/cmdline | grep -o "zswap.enabled=1 zswap.compressor=zstd"
```

### 5. Monitor SSD Wear & Health
```bash
sudo apt install smartmontools
sudo smartctl -a /dev/sda | grep -iE "percentage|wearout|total_lba"
```

---

## ⚠️ Important Notes
- **Never reboot** after editing `/etc/fstab` without running `findmnt --verify` first.
- **Avoid `discard` in fstab:** Scheduled TRIM (`fstrim.timer`) is safer, more efficient, and aligns better with SSD controller garbage collection.
- **NVMe Power Latency:** Only use `nvme_core.default_ps_max_latency_us=0` if you experience stuttering. Otherwise, let the drive manage its own power states.
- **Backup Strategy:** Keep Timeshift snapshots active until you've validated the system for a few days.
