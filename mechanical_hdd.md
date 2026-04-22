# Linux Mint HDD Optimization Guide

## Use this configuration guide for mechanical HDDs only...NOT SSDs or NVMe

## 1. Apply Hardened Sysctl Configuration
Copy the pre-configured hardening file to the system directory:
```bash
sudo cp HDD_99-mint-hardening.conf /etc/sysctl.d/99-mint-hardening.conf
```

## 2. Configure `bfq` I/O Scheduler (HDD-Optimized)
> Reduces mechanical head-seeking thrashing during multitasking.

### Step 1: Auto-detect HDD & Create udev Rule
```bash
HDD=$(lsblk -d -o NAME,ROTA | awk '$2==1 {print $1}' | head -1)
echo "ACTION==\"add|change\", KERNEL==\"${HDD}\", ATTR{queue/scheduler}=\"bfq\"" | sudo tee /etc/udev/rules.d/60-io-scheduler.rules
```
*(Note: If the variable returns empty due to controller quirks, manually set it: `HDD="sda"`)*

### Step 2: Apply Immediately
```bash
echo bfq | sudo tee /sys/block/${HDD}/queue/scheduler
```

### Step 3: Verify It's Active
```bash
cat /sys/block/${HDD}/queue/scheduler
```
✅ **Expected output:** `mq-deadline kyber [bfq] none`  
*(The `[brackets]` indicate the currently active scheduler.)*

### 🔄 Rollback (If Needed)
```bash
sudo rm /etc/udev/rules.d/60-io-scheduler.rules
echo mq-deadline | sudo tee /sys/block/${HDD}/queue/scheduler
```

## 3. Optimize `/etc/fstab` for HDD Performance
> Cuts unnecessary metadata writes by ~30% and batches disk flushes for smoother HDD performance.

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
/dev/mapper/vgmint-root /  ext4  defaults,noatime,commit=60,errors=remount-ro  0  1
# or
UUID=####################  /  ext4  defaults,noatime,commit=60,errors=remount-ro  0  1
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

## 4. Enable `zswap` via GRUB
Compresses swapped pages in RAM before writing to the mechanical drive, drastically reducing swap-induced freezes.

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
Change it to:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash zswap.enabled=1 zswap.compressor=lz4"
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

### 2. Check zswap Status
```bash
dmesg | grep zswap
```
✅ **Expected:** `"zswap: loaded using pool lz4/zsmalloc"`

```bash
cat /sys/module/zswap/parameters/enabled
```
✅ **Should output:** `Y`

### 3. Verify Sysctl Persistence
```bash
sysctl -p /etc/sysctl.d/99-mint-hardening.conf 2>&1 | grep -i error
```

### 4. Check I/O Scheduler
```bash
cat /sys/block/$(lsblk -d -o NAME,ROTA | awk '$2==1 {print $1}' | head -1)/queue/scheduler
```

### 5. Verify Mount Options (Detailed)
```bash
mount | grep " / " | grep -o "noatime.*commit=60"
```

### 6. Monitor I/O Wait
```bash
vmstat 1 5 | awk 'NR>2 {print $16"% wa"}'
```
✅ **Target:** Should stay `<5%` during normal desktop use.

---

## ⚠️ Important Notes
- **Never reboot** after editing `/etc/fstab` without running `findmnt --verify` first.
- **Dual-booting Windows?** Disable "Fast Startup" in Windows to prevent filesystem locks on shared drives.
- **Backup Strategy:** Keep your backup drive connected for Timeshift until you've tested everything for a few days.
