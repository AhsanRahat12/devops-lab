# How to Setup Raspberry Pi with YubiKey FIDO2 SSH Authentication

## Overview
Complete guide for passwordless SSH authentication to Raspberry Pi devices using YubiKey FIDO2 hardware security keys. This setup eliminates password vulnerabilities and requires physical key presence for all connections.

---

## Table of Contents
1. [What is FIDO2 SSH Authentication?](#what-is-fido2-ssh-authentication)
2. [Prerequisites](#prerequisites)
3. [Setup Process](#setup-process)
4. [SSH Configuration](#ssh-configuration)
5. [Hardening SSH](#hardening-ssh)
6. [Adding New Devices](#adding-new-devices)
7. [Daily Usage](#daily-usage)
8. [Troubleshooting](#troubleshooting)
9. [Security Considerations](#security-considerations)

---

## What is FIDO2 SSH Authentication?

**FIDO2** (Fast IDentity Online 2) enables hardware-based authentication for SSH:

- **Private key stays on hardware** - Cannot be extracted, even if computer is compromised
- **Physical touch required** - Proof of presence for each authentication
- **Phishing resistant** - Hardware-bound cryptographic authentication
- **OpenSSH 8.2+ native support**
- **Passwordless** - Eliminates password vulnerabilities

**Key Type:** `sk-ssh-ed25519@openssh.com`
- `sk-` = "security key" (FIDO2/U2F)
- `ed25519` = Elliptic curve algorithm
- `@openssh.com` = OpenSSH implementation

**Traditional vs FIDO2:**
- Traditional: Private key stored as file (can be copied/stolen)
- FIDO2: Private key stored IN hardware (cannot be extracted)

---

## Prerequisites

### System Requirements

```bash
# Verify OpenSSH version (8.2+ required)
ssh -V

# Install FIDO2 support
sudo pacman -S libfido2

# Verify YubiKey detection
lsusb | grep -i yubikey
```

### Hardware Requirements
- Two YubiKeys with FIDO2 support (YubiKey 5 series or later)
- Raspberry Pi running Linux
- Network access (LAN or VPN)

---

## Setup Process

### Step 1: Generate FIDO2 SSH Keys

**⚠️ CRITICAL: YubiKey must be plugged in during generation!**

#### Primary Key
```bash
# Plug in PRIMARY YubiKey
ssh-keygen -t ed25519-sk -C "primary-yubikey" -f ~/.ssh/id_ed25519_sk_primary
```

**During generation:**
- YubiKey LED will blink
- Touch the metal contact when prompted
- Press Enter for passphrase (optional)

**Files created:**
- `~/.ssh/id_ed25519_sk_primary` - Key reference
- `~/.ssh/id_ed25519_sk_primary.pub` - Public key

#### Backup Key
```bash
# Remove primary, plug in BACKUP YubiKey
ssh-keygen -t ed25519-sk -C "backup-yubikey" -f ~/.ssh/id_ed25519_sk_backup
```

#### Verify Generation
```bash
# List keys
ls -la ~/.ssh/id_*

# Verify FIDO2 format
cat ~/.ssh/id_*.pub
# Should show: sk-ssh-ed25519@openssh.com
```

---

### Step 2: Copy Keys to Raspberry Pi

**⚠️ CRITICAL: Copy BOTH keys to EACH Pi for redundancy!**

```bash
# Copy PRIMARY key
ssh-copy-id -i ~/.ssh/id_ed25519_sk_primary.pub pi@DEVICE_IP

# Copy BACKUP key
ssh-copy-id -i ~/.ssh/id_ed25519_sk_backup.pub pi@DEVICE_IP
```

#### Verify Both Keys Added
```bash
# SSH into Pi
ssh pi@DEVICE_IP

# Check authorized_keys
cat ~/.ssh/authorized_keys
```

**Expected output - TWO LINES:**
```
sk-ssh-ed25519@openssh.com AAAAGnNr... primary-yubikey
sk-ssh-ed25519@openssh.com AAAAGnNr... backup-yubikey
```

✅ Two lines present = Correct
❌ One line = Missing key, go back and copy it!

```bash
exit
```

---

### Step 3: Configure SSH Client

```bash
nano ~/.ssh/config
```

**Add configuration:**
```
Host pi-device
    HostName DEVICE_IP
    User pi
    IdentityFile ~/.ssh/id_ed25519_sk_primary
    IdentityFile ~/.ssh/id_ed25519_sk_backup
    PreferredAuthentications publickey
    PasswordAuthentication no
```

**Set permissions:**
```bash
chmod 600 ~/.ssh/config
```

---

### Step 4: Test Authentication

#### Test Primary Key
```bash
# Plug in PRIMARY YubiKey
ssh pi-device
# Touch YubiKey when it blinks
# Should connect without password
exit
```

#### Test Backup Key
```bash
# Remove primary, plug in BACKUP YubiKey
ssh pi-device
# Touch backup YubiKey when it blinks
# Should connect without password
exit
```

✅ Both work = Proceed to hardening
❌ Either fails = Troubleshoot before continuing

---

## Hardening SSH

**⚠️ WARNING: Only proceed after confirming BOTH YubiKeys work!**

### On Raspberry Pi

```bash
# SSH into Pi
ssh pi-device

# Backup config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Edit config
sudo nano /etc/ssh/sshd_config
```

**Modify these settings:**
```
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
```

**Restart SSH:**
```bash
sudo systemctl restart sshd
```

### Test Before Closing Connection

**Open NEW terminal:**
```bash
# Test YubiKey works
ssh pi-device
# Should work ✅

# Test password disabled
ssh -o PreferredAuthentications=password pi@DEVICE_IP
# Should fail with "Permission denied (publickey)" ✅
```

---

## Adding New Devices

### Quick Checklist
```
□ Get device IP address
□ Initial password SSH access
□ Copy PRIMARY key
□ Copy BACKUP key
□ Verify BOTH keys in authorized_keys
□ Add to ~/.ssh/config
□ Test PRIMARY YubiKey
□ Test BACKUP YubiKey
□ Harden SSH
□ Final verification
```

### Process

```bash
# 1. Copy BOTH keys
ssh-copy-id -i ~/.ssh/id_ed25519_sk_primary.pub pi@NEW_DEVICE_IP
ssh-copy-id -i ~/.ssh/id_ed25519_sk_backup.pub pi@NEW_DEVICE_IP

# 2. Verify both present
ssh pi@NEW_DEVICE_IP
cat ~/.ssh/authorized_keys
# Must show TWO lines!
exit

# 3. Add to SSH config
nano ~/.ssh/config
# Add Host entry with BOTH IdentityFile lines

# 4. Test both YubiKeys before hardening
```

---

## Daily Usage

### Basic Connection
```bash
# Plug in YubiKey (primary or backup)
ssh pi-device
# Touch YubiKey when it blinks
# Connected!
```

### Multiple Sessions
```bash
# Each session requires YubiKey touch
ssh pi-device  # Terminal 1
ssh pi-device  # Terminal 2
```

---

## Troubleshooting

### "sign_and_send_pubkey: signing failed"
**Cause:** YubiKey not detected
```bash
# Check detection
lsusb | grep -i yubikey
# Try different USB port
# Replug YubiKey
```

### "Permission denied (publickey)" with backup key
**Cause:** Backup key missing from authorized_keys
```bash
# Verify keys on Pi
ssh -o PreferredAuthentications=password pi@DEVICE_IP
cat ~/.ssh/authorized_keys
# Should show TWO lines

# If missing, copy backup key
exit
ssh-copy-id -i ~/.ssh/id_ed25519_sk_backup.pub pi@DEVICE_IP
```

### YubiKey blinks but touch doesn't work
**Solution:**
- Touch metal contact firmly
- Hold 1-2 seconds
- Ensure full USB insertion

### Locked out after disabling passwords
**Physical access solution:**
```bash
# Connect keyboard/monitor to Pi
# Log in locally
sudo cp /etc/ssh/sshd_config.backup /etc/ssh/sshd_config
sudo systemctl restart sshd
```

### Debug Connection
```bash
# Verbose output
ssh -v pi-device

# Very verbose
ssh -vvv pi-device
```

---

## Security Considerations

### Protection Provided

| Attack Type | Protected? |
|-------------|-----------|
| Password brute force | ✅ Yes |
| Stolen laptop | ✅ Yes |
| Malware key theft | ✅ Yes |
| Phishing | ✅ Yes |
| Remote exploitation | ✅ Yes |
| Replay attacks | ✅ Yes |

### Best Practices

✅ **Do:**
- Keep backup YubiKey in separate location
- Set YubiKey PIN for extra security
- Regular system updates
- Monitor SSH logs: `sudo journalctl -u ssh`
- Keep `sshd_config.backup` files
- Verify BOTH keys on ALL devices

❌ **Don't:**
- Share YubiKeys
- Leave YubiKey plugged in when unused
- Skip testing before hardening
- Add unknown keys to authorized_keys

### Emergency Recovery

**If both YubiKeys lost:**
1. Physical access: Connect keyboard/monitor, log in locally
2. SD card method: Mount on another computer, edit sshd_config
3. Prevention: Keep backup YubiKey secure and separate

---

## Understanding Dual-Key Setup

### Authentication Flow

```
Your Machine                  Raspberry Pi
─────────────                 ────────────
SSH Config:                   authorized_keys:
├── IdentityFile primary ───→ Line 1: primary key
└── IdentityFile backup  ───→ Line 2: backup key

With PRIMARY plugged in:
SSH tries primary → matches Line 1 ✅

With BACKUP plugged in:
SSH tries primary → fails (not present)
SSH tries backup  → matches Line 2 ✅

With NO YubiKey:
SSH tries primary → fails
SSH tries backup  → fails
Result: Permission denied ❌
```

### Why Both Keys in Both Places?

**On Pi's authorized_keys:**
- Contains public keys allowed to authenticate
- Need BOTH lines for true redundancy

**In SSH config:**
- Contains references to try
- Need BOTH IdentityFile lines for automatic fallback

**Both must be configured for redundancy!**

---

## Quick Reference

### Key Generation
```bash
# Primary key
ssh-keygen -t ed25519-sk -C "primary-yubikey" -f ~/.ssh/id_ed25519_sk_primary

# Backup key
ssh-keygen -t ed25519-sk -C "backup-yubikey" -f ~/.ssh/id_ed25519_sk_backup
```

### Key Distribution
```bash
# Copy BOTH keys (critical!)
ssh-copy-id -i ~/.ssh/id_ed25519_sk_primary.pub pi@DEVICE_IP
ssh-copy-id -i ~/.ssh/id_ed25519_sk_backup.pub pi@DEVICE_IP

# Verify
ssh pi@DEVICE_IP
cat ~/.ssh/authorized_keys
# Must show TWO lines!
```

### Connection
```bash
ssh pi-device          # Connect
ssh -v pi-device       # Debug
ssh -vvv pi-device     # Verbose debug
```

### Maintenance
```bash
# Backup config
cp ~/.ssh/config ~/.ssh/config.backup

# Restore on Pi
sudo cp /etc/ssh/sshd_config.backup /etc/ssh/sshd_config
sudo systemctl restart sshd
```

---

## Additional Resources

- [OpenSSH FIDO/U2F Documentation](https://www.openssh.com/txt/release-8.2)
- [YubiKey Manager CLI Guide](https://docs.yubico.com/software/yubikey/tools/ykman/)
- [Arch Linux SSH Keys Guide](https://wiki.archlinux.org/title/SSH_keys)
- [FIDO Alliance Specifications](https://fidoalliance.org/specifications/)

---

**Document Version:** 1.1  
**Last Updated:** 2026-01-18
