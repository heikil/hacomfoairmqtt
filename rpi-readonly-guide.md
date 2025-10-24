# Raspberry Pi Zero W Read-Only Filesystem Guide

This guide covers converting your Raspberry Pi Zero W to a read-only filesystem using OverlayFS to protect the SD card and ensure the system reverts to its initial state on reboot.

## Understanding Read-Only Protection

There are **two separate protection mechanisms**:

### 1. OverlayFS (File System Level)
```bash
raspi-config nonint enable_overlayfs
raspi-config nonint disable_overlayfs
```
- Creates a RAM overlay over your filesystem
- All writes go to RAM, original SD card remains untouched
- Changes are lost on reboot
- **Persists across reboots** until explicitly disabled

### 2. Boot Partition Read-Only (Boot Partition Level)
```bash
raspi-config nonint enable_bootro
raspi-config nonint disable_bootro
```
- Makes only the `/boot` partition read-only
- Protects boot files (config.txt, cmdline.txt, kernel, etc.)
- Less comprehensive than overlayfs

**Recommendation:** Enable both for maximum protection.

## Pre-Conversion Checklist

Complete these steps **before** enabling read-only mode:

### 1. Verify Service is Working
```bash
sudo systemctl status ca350.service
sudo journalctl -u ca350.service -n 50
```
Ensure everything runs perfectly - fixes are harder in read-only mode.

### 2. Check for File Writes
```bash
# Check what the service is writing
sudo lsof +D /home/h/hacomfoair 2>/dev/null | grep -v ".pyc"
```

### 3. Configure Logging to RAM

Edit the ca350 service:
```bash
sudo nano /etc/systemd/system/ca350.service
```

Add these lines under `[Service]`:
```ini
StandardOutput=journal
StandardError=journal
```

Configure journald for volatile storage:
```bash
sudo nano /etc/systemd/journald.conf
```

Uncomment and set:
```ini
Storage=volatile
RuntimeMaxUse=10M
```

### 4. Disable Swap
```bash
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo systemctl disable dphys-swapfile
```

### 5. Disable Unnecessary Services
```bash
# Reduce disk writes
sudo systemctl disable apt-daily.timer
sudo systemctl disable apt-daily-upgrade.timer
sudo systemctl disable man-db.timer
```

### 6. Setup Automatic Restarts

For twice-daily restarts (6 AM and 6 PM):
```bash
sudo nano /etc/crontab
```

Add:
```
0 6,18 * * * root /sbin/shutdown -r +1
```

### 7. Backup Your SD Card

**CRITICAL:** Create a backup before enabling read-only mode.

On your computer (Linux/Mac):
```bash
# Find SD card device (e.g., /dev/sdb or /dev/mmcblk0)
lsblk

# Create backup image
sudo dd if=/dev/sdX of=~/pi-backup-$(date +%Y%m%d).img bs=4M status=progress
```

On Windows, use Win32DiskImager or similar tool.

## Enabling Read-Only Mode

Run as root:
```bash
sudo raspi-config nonint enable_overlayfs
sudo raspi-config nonint enable_bootro
sudo reboot
```

## Verifying Read-Only Mode

After reboot, test:
```bash
# This should work (tmpfs is writable)
touch /tmp/test

# This should fail with "Read-only file system" error
touch ~/test

# Check overlay status
mount | grep overlay

# Verify service is running
sudo systemctl status ca350.service
```

## Making Changes After Enabling Read-Only

When you need to update or modify the system:

### Step 1: Disable Read-Only Mode
```bash
sudo raspi-config nonint disable_overlayfs
sudo raspi-config nonint disable_bootro
```

### Step 2: Reboot to Writable Mode
```bash
sudo reboot
```

### Step 3: Make Your Changes
```bash
# Update packages, edit configs, etc.
# Example:
cd ~/hacomfoair
source .venv/bin/activate
pip install some-package

# Or edit config
nano ~/hacomfoair/src/config.ini
```

### Step 4: Re-enable Read-Only Mode
```bash
sudo raspi-config nonint enable_overlayfs
sudo raspi-config nonint enable_bootro
```

### Step 5: Reboot Back to Read-Only
```bash
sudo reboot
```

## Troubleshooting

### Service Not Starting After Read-Only Enabled

Check logs:
```bash
sudo journalctl -u ca350.service -f
```

Common issues:
- Service trying to write log files to disk
- Config files in wrong location
- Permission issues

### Out of Memory Errors

Pi Zero W has only 512MB RAM. The overlay uses RAM for writes.

Check memory usage:
```bash
free -h
top
```

If needed, reduce RuntimeMaxUse in journald.conf.

### Need Emergency Access

Boot to recovery mode or remove SD card and edit on another computer.

## Performance Considerations

**Pi Zero W Limitations:**
- 512MB RAM (overlay uses some for temporary writes)
- Single-core CPU (slight overhead from overlay)
- Monitor memory usage regularly

**Expected Behavior:**
- System always reverts to clean state on reboot
- Twice-daily restarts clear memory and temporary files
- SD card lifespan significantly extended

## Quick Reference Commands

```bash
# Enable read-only
sudo raspi-config nonint enable_overlayfs
sudo raspi-config nonint enable_bootro
sudo reboot

# Disable read-only
sudo raspi-config nonint disable_overlayfs
sudo raspi-config nonint disable_bootro
sudo reboot

# Check overlay status
mount | grep overlay

# Check memory usage
free -h

# View service logs
sudo journalctl -u ca350.service -f

# Force reboot
sudo reboot
```

## Notes

- All changes are **persistent** until you disable them
- OverlayFS persists across reboots - not a one-time writable boot
- Always backup before enabling read-only mode
- Test thoroughly before deploying
- Document any custom configurations for future reference