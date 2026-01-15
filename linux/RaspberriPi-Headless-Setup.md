# Complete Raspberry Pi Headless Setup Guide

A comprehensive guide for setting up a Raspberry Pi for headless operation using an SSD, with hardware-backed SSH authentication via YubiKey.

**My Testing Configuration:**
- Raspberry Pi 5
- 32GB SSD
- Raspberry Pi OS Bookworm (64-bit)
- Ubuntu/Arch Linux host system
- Dual YubiKey setup (primary + backup)

**Note:** While this guide was tested with the above hardware, the process should work with any Raspberry Pi model and SSD size (minimum 8GB for Raspberry Pi OS Lite, 16GB for full desktop version).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Part 1: Preparing the SSD](#part-1-preparing-the-ssd)
3. [Part 2: First Boot and Network Configuration](#part-2-first-boot-and-network-configuration)
4. [Part 3: SSH Key Setup with YubiKeys](#part-3-ssh-key-setup-with-yubikeys)
5. [Part 4: System Maintenance Setup](#part-4-system-maintenance-setup)
6. [Troubleshooting](#troubleshooting)
7. [Next Steps](#next-steps)
8. [Additional Resources](#additional-resources)
9. [Security Notes](#security-notes)

---

## Prerequisites

### Software Required
- Linux host system (Ubuntu/Arch/Debian)
- Raspberry Pi Imager
- `nmap` for network scanning
- OpenSSH with FIDO2 support (for YubiKey authentication)

---

## Part 1: Preparing the SSD

### Critical: Why Manual Configuration No Longer Works

**The traditional method of creating `wpa_supplicant.conf` and `ssh` files manually is deprecated and does not work with modern Raspberry Pi OS (Bookworm and later).** You must use Raspberry Pi Imager's built-in configuration system.

### Step 1: Install Raspberry Pi Imager
```bash
sudo apt update
sudo apt install rpi-imager
```

### Step 2: Clean the SSD

Skip this section if using a new, never-configured SSD.

#### Option A: Use Pi Imager's Erase Feature

1. Launch Raspberry Pi Imager:
```bash
   rpi-imager
```

2. Select "CHOOSE OS" → Scroll to bottom → "Erase"

3. Select "CHOOSE STORAGE" → Your SSD

4. Click "WRITE"

If you encounter `"Error synchronizing after initial wipe: Timed out waiting for object"`, proceed to Option B.

#### Option B: Manual Cleanup (When Erase Fails)

1. **Identify your SSD:**
```bash
   lsblk
```
   Look for your device (typically `/dev/sda` or `/dev/sdb` with matching size)

2. **Unmount all partitions:**
```bash
   sudo umount /dev/sdX* 2>/dev/null
```
   Replace `sdX` with your actual device identifier.

3. **Zero out the partition table:**
```bash
   sudo dd if=/dev/zero of=/dev/sdX bs=1M count=100 status=progress
```
   This writes 100MB of zeros and takes approximately 3-5 seconds.

4. **Create fresh partition table:**
```bash
   sudo fdisk /dev/sdX
```
   - Press `o` + Enter (creates new DOS partition table)
   - Press `w` + Enter (writes changes and exits)

5. **Verify clean state:**
```bash
   lsblk
```
   You should see only your device with no child partitions.

### Step 3: Install Pi OS with Embedded Configuration

1. **Launch Pi Imager:**
```bash
   rpi-imager
```

2. **Choose Operating System:**
   - Click "CHOOSE OS"
   - Select "Raspberry Pi OS (64-bit)" or "Raspberry Pi OS Lite (64-bit)" for headless setups
   - Pi Imager will download automatically (~1.2GB for full OS, ~500MB for Lite)

3. **Select Storage:**
   - Click "CHOOSE STORAGE"
   - Select your SSD

4. **Configure Advanced Settings (Critical!):**

   Click the **gear icon (⚙️)** in the bottom right to open settings:

   **General Tab:**
   - ☑ **Set hostname:** Choose a meaningful name (e.g., `pi-homelab`, `raspberrypi`)
   - ☑ **Set username and password:**
     - Username: Your choice (default is `pi`)
     - Password: Your secure password (save this!)

   **Services Tab:**
   - ☑ **Enable SSH**
     - Select "Use password authentication"

   **Wireless LAN Tab:**
   - ☑ **Configure wireless LAN:**
     - SSID: Your WiFi network name
     - Password: Your WiFi password
     - Wireless LAN country: Your country code (e.g., `US`, `CA`, `GB`)

   **Locale Tab:**
   - ☑ **Set locale settings:**
     - Time zone: Your timezone (e.g., `America/New_York`, `Europe/London`)
     - Keyboard layout: Your layout (e.g., `us`, `gb`)

   **Options Tab:**
   - ☑ **Eject media when finished** (recommended)
   - Other options are personal preference

5. **Click "SAVE"** to preserve all settings

6. **Click "WRITE"** to begin installation

   The process includes:
   - Downloading OS image
   - Writing to SSD
   - Embedding your configuration
   - Verification
   
   **Expected duration:** 10-20 minutes

7. **Wait for completion** and safe eject notification

---

## Part 2: First Boot and Network Configuration

### Step 1: Initial Boot

1. Insert SSD into Raspberry Pi

2. Connect power

3. Wait 3-5 minutes for first boot sequence:
   - Filesystem expansion
   - WiFi connection
   - SSH service initialization
   - Configuration application

### Step 2: Locate Your Pi on the Network

**Method Used: Router Admin Panel**
- Access router (typically `192.168.1.1` or `192.168.0.1`)
- Navigate to DHCP clients or connected devices
- Locate device by hostname

### Step 3: Configure DHCP Reservation

Ensures consistent IP address across reboots.

1. Access your router's admin panel

2. Navigate to DHCP settings (location varies by router)

3. Create new reservation:
   - **Hostname:** Your Pi's hostname
   - **MAC Address:** Find in router's device list or via:
```bash
     ssh username@<current-ip>
     ip link show
```
   - **Reserved IP:** Choose static address in your network range (e.g., `192.168.1.100`)

4. Save configuration

5. Reboot Pi to apply:
```bash
   ssh username@<current-ip>
   sudo reboot
```

### Step 4: Verify SSH Access

Test connection with reserved IP:
```bash
ssh username@<reserved-ip>
```

You should authenticate with the password configured in Pi Imager.

---

## Part 3: SSH Key Setup with YubiKeys

Hardware-based authentication provides superior security and convenience.

### Prerequisites Check

Verify FIDO2 support in OpenSSH:
```bash
ssh-keygen -t ed25519-sk
```

If this fails, update OpenSSH or install `libfido2`.

### Step 1: Generate Primary YubiKey SSH Key

1. **Insert primary YubiKey**

2. **Generate resident key:**
```bash
   ssh-keygen -t ed25519-sk -O resident -C "primary-yubikey" -f ~/.ssh/id_ed25519_sk_primary
```

   **Parameters explained:**
   - `-t ed25519-sk`: Ed25519 algorithm with security key
   - `-O resident`: Store key on YubiKey (enables portability)
   - `-C "primary-yubikey"`: Key identifier comment
   - `-f ~/.ssh/id_ed25519_sk_primary`: Output file path

3. **Follow prompts:**
   - Touch YubiKey to confirm
   - Enter YubiKey PIN (if configured)
   - Set passphrase (optional, recommended for additional security)

### Step 2: Generate Backup YubiKey SSH Key

1. **Remove primary YubiKey**

2. **Insert backup YubiKey**

3. **Generate backup key:**
```bash
   ssh-keygen -t ed25519-sk -O resident -C "backup-yubikey" -f ~/.ssh/id_ed25519_sk_backup
```

4. **Complete same prompts as primary key**

### Step 3: Deploy Public Keys to Pi

1. **Copy primary key:**
```bash
   ssh-copy-id -i ~/.ssh/id_ed25519_sk_primary.pub username@<pi-ip-address>
```
   Enter Pi password when prompted.

2. **Copy backup key:**
```bash
   ssh-copy-id -i ~/.ssh/id_ed25519_sk_backup.pub username@<pi-ip-address>
```
   Enter Pi password again.

Both public keys are now in `~/.ssh/authorized_keys` on your Pi.

### Step 4: Configure SSH Client

Create or edit `~/.ssh/config` on your host machine:
```bash
vim ~/.ssh/config
```

Add configuration:
```
Host pi-server
    HostName <pi-ip-address>
    User <username>
    IdentityFile ~/.ssh/id_ed25519_sk_primary
    PreferredAuthentications publickey
```

**Configuration breakdown:**
- `Host`: Connection alias (choose any name you like)
- `HostName`: Pi's reserved IP address
- `User`: Your username on the Pi
- `IdentityFile`: Path to primary YubiKey private key
- `PreferredAuthentications`: Prioritize key authentication

Save and close (`:wq` in vim).

### Step 5: Test Primary YubiKey Authentication

1. **Ensure primary YubiKey is connected**

2. **Connect using alias:**
```bash
   ssh pi-server
```

3. **Expected prompts:**
   - Touch YubiKey to authenticate
   - Enter YubiKey PIN (if configured)

4. **Success:** You connect without Pi password

### Step 6: Verify Backup YubiKey

1. **Remove primary YubiKey**

2. **Insert backup YubiKey**

3. **Connect specifying backup key:**
```bash
   ssh -i ~/.ssh/id_ed25519_sk_backup username@<pi-ip-address>
```

Backup authentication should succeed.

### Optional: Disable Password Authentication

**⚠️ Warning:** Only proceed after confirming both YubiKeys work successfully.

1. **SSH into Pi**

2. **Edit SSH daemon configuration:**
```bash
   sudo vim /etc/ssh/sshd_config
```

3. **Modify these directives:**
```
   PasswordAuthentication no
   PubkeyAuthentication yes
   ChallengeResponseAuthentication no
```

4. **Restart SSH service:**
```bash
   sudo systemctl restart sshd
```

Your Pi now requires key-based authentication exclusively.

---

## Part 4: System Maintenance Setup

Automate security updates and necessary reboots.

### Step 1: Create Update Script

1. **SSH into Pi:**
```bash
   ssh pi-server
```

2. **Create maintenance script:**
```bash
   vim ~/update-and-reboot.sh
```

3. **Add script content:**
```bash
   #!/bin/bash
   
   # Update package database and upgrade all packages
   sudo apt update && sudo apt upgrade -y
   
   # Reboot if system updates require it
   if [ -f /var/run/reboot-required ]; then
       sudo reboot
   fi
```

4. **Save and exit** (`:wq`)

5. **Make executable:**
```bash
   chmod +x ~/update-and-reboot.sh
```

### Step 2: Test Script

Execute manually to verify functionality:
```bash
~/update-and-reboot.sh
```

The script will:
- Update package lists
- Upgrade installed packages
- Automatically reboot if `/var/run/reboot-required` exists

Your DHCP reservation ensures the same IP after reboot.

### Step 3: Schedule Automatic Execution

1. **Open crontab editor:**
```bash
   crontab -e
```

2. **Add scheduled task:**
```
   0 7 * * * /home/<username>/update-and-reboot.sh
```
   Replace `<username>` with your actual username.

   **Cron syntax breakdown:**
```
   ┌─── minute (0-59)
   │ ┌─── hour (0-23)
   │ │ ┌─── day of month (1-31)
   │ │ │ ┌─── month (1-12)
   │ │ │ │ ┌─── day of week (0-7, 0 and 7 = Sunday)
   │ │ │ │ │
   0 7 * * *
```
   
   **This schedule:** Daily at 7:00 AM

3. **Save and close**

4. **Verify cron job:**
```bash
   crontab -l
```

### Understanding Reboot Behavior

**Reboot triggers:**
- Kernel updates
- Firmware updates
- Core system library updates

**No reboot needed for:**
- Application updates
- User-space package updates
- Configuration file changes

The system automatically creates `/var/run/reboot-required` when necessary.

### Alternative Schedules

**Weekly (Sunday 3:00 AM):**
```
0 3 * * 0 /home/<username>/update-and-reboot.sh
```

**Monthly (First day, 2:00 AM):**
```
0 2 1 * * /home/<username>/update-and-reboot.sh
```

**Every 6 hours:**
```
0 */6 * * * /home/<username>/update-and-reboot.sh
```

---

## Troubleshooting

### YubiKey Authentication Fails

**Symptoms:** Key authentication not working.

**Solutions:**

1. Verify YubiKey is properly seated in USB port
2. Confirm FIDO2 support:
```bash
   ssh -V  # Check OpenSSH version
   ssh-keygen -t ed25519-sk  # Test FIDO2
```
3. Verify public key deployment:
```bash
   ssh username@<pi-ip> "cat ~/.ssh/authorized_keys"
```
4. Check for PIN lockout (too many failed attempts)
5. Test with verbose output:
```bash
   ssh -v username@<pi-ip>
```

### IP Address Changes After Reboot

**Symptoms:** DHCP reservation not persisting.

**Solutions:**

1. Verify MAC address in DHCP reservation matches Pi
2. Check reservation is saved in router configuration
3. Alternative: Static IP on Pi directly
```bash
   sudo vim /etc/dhcpcd.conf
```
   Add:
```
   interface wlan0
   static ip_address=<desired-ip>/24
   static routers=<router-ip>
   static domain_name_servers=<dns-server-ip>
```
   Then reboot:
```bash
   sudo reboot
```

### Permission Denied Errors

**Symptoms:** Cannot edit files or execute commands.

**Solutions:**

1. Use `sudo` for system files:
```bash
   sudo vim /etc/ssh/sshd_config
```

2. Fix home directory ownership:
```bash
   sudo chown -R username:username /home/username/
```

3. Check file permissions:
```bash
   ls -la /path/to/file
```

---

Raspberry Pi is now configured with:

- ✅ Headless operation (no monitor/keyboard required)
- ✅ Automatic WiFi connectivity
- ✅ Hardware-backed SSH authentication
- ✅ Consistent network addressing
- ✅ Automated security maintenance
- ✅ Redundant authentication method

## Additional Resources

- [Official Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)
- [Raspberry Pi Forums](https://forums.raspberrypi.com/)
- [YubiKey SSH Guide](https://developers.yubico.com/SSH/)
- [OpenSSH Configuration](https://www.ssh.com/academy/ssh/config)
- [Cron Syntax Reference](https://crontab.guru/)

---

## Security Notes

**Important Security Practices:**

- Never share your YubiKey PINs
- Keep backup YubiKey in a secure, separate location
- Regularly update your Pi's software
- Monitor SSH logs for unauthorized access attempts:
```bash
  sudo journalctl -u sshd -f
```
- Consider implementing fail2ban for additional security:
```bash
  sudo apt install fail2ban
```
- Regularly backup important data
- Change default passwords immediately
- Disable unused services

---

**Guide Information:**
- **Version:** 2.1
- **Last Updated:** January 2024
- **My Testing Platform:** Raspberry Pi 5 with 32GB SSD
- **Host OS:** Ubuntu 24.04 / Arch Linux
- **Pi OS:** Bookworm (64-bit)
- **Minimum Storage:** 8GB for Lite, 16GB for Desktop version

---

## License

This guide is shared for educational purposes. Feel free to use, modify, and distribute with attribution.

## Contributing

Found an issue or have an improvement? Feel free to open an issue or submit a pull request.
