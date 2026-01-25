# WiFi Disconnection Troubleshooting - Arch Linux

## Problem Description
Laptop completely lost WiFi connectivity and could not reconnect. Browser and system showed "Can't connect to WiFi/Internet" with no automatic reconnection occurring.

## Initial Symptoms
- Complete WiFi connection failure
- System appeared to be in airplane mode (though it wasn't)
- Browser showing "No internet" - not connecting at all
- No automatic reconnection happening

## Environment
- **OS**: Arch Linux
- **Window Manager**: Hyprland
- **Network Manager**: systemd-networkd (NOT NetworkManager)
- **WiFi Interface**: wlan0
- **Access Point**: "Rahat & Tori 5G"

## Diagnostic Steps

### 1. Check Airplane Mode Status
```bash
rfkill list
```
**Result**: All wireless devices showed "Soft blocked: no" and "Hard blocked: no" - airplane mode was NOT the issue.

### 2. Check NetworkManager Status
```bash
systemctl status NetworkManager
```
**Result**: `Unit NetworkManager.service could not be found` - system was using systemd-networkd instead.

### 3. Check systemd-networkd Status
```bash
systemctl status systemd-networkd
```
**Result**: Service was active and running.

### 4. Initial Resolution - System Restart
A system restart temporarily restored connectivity, but we investigated logs to find root cause.

### 5. Analyze System Logs (Previous Boot)
```bash
journalctl -u systemd-networkd -b -1
```
**Key Finding**: Pattern of disconnects throughout the day BEFORE the final failure:
```
wlan0: Lost carrier
wlan0: Connected WiFi access point: Rahat & Tori 5G
wlan0: Gained carrier
```

Disconnect timestamps:
- 19:20, 19:24, 19:35, 19:41
- 20:38, 20:54, 20:58  
- 21:39 (twice)
- 23:04 (final failure - no automatic recovery)

### 6. Install Wireless Tools
```bash
sudo pacman -S iw
```

### 7. Check WiFi Power Save Status
```bash
iw dev wlan0 get power_save
```
**Result**: `Power save: on` ← **ROOT CAUSE IDENTIFIED**

## Root Cause
WiFi power management was enabled by default in the Linux kernel. Throughout the day, it caused brief disconnections that auto-recovered. Eventually, one of these power-save cycles failed to reconnect, resulting in complete connection loss that required a manual restart to recover.

## Solution Implemented

### Temporary Fix (Current Session Only)
```bash
sudo iw dev wlan0 set power_save off
```

### Permanent Fix (Survives Reboots)

**1. Create systemd service file:**
```bash
sudo nvim /etc/systemd/system/wifi-power-save-off.service
```

**2. Add service configuration:**
```ini
[Unit]
Description=Disable WiFi Power Management
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/iw dev wlan0 set power_save off

[Install]
WantedBy=multi-user.target
```

**3. Enable and start the service:**
```bash
sudo systemctl enable wifi-power-save-off.service
sudo systemctl start wifi-power-save-off.service
```

## Verification

**1. Check service status:**
```bash
systemctl status wifi-power-save-off.service
```
Expected: `Active: active (exited)` with no errors

**2. Verify power save is disabled:**
```bash
iw dev wlan0 get power_save
```
Expected: `Power save: off`

**3. Monitor network stability:**
```bash
journalctl -u systemd-networkd -f
```
Watch for absence of "Lost carrier" messages

## Results
- WiFi connection stabilized after disabling power save
- No more connection failures
- Trade-off: Slight increase in power consumption (~5-10 min battery life over several hours)

## Key Takeaways
1. **Logs reveal hidden issues**: Even though the user only noticed complete connection failure, logs showed power management was causing problems throughout the day
2. **Power save can cause complete failures**: Not just brief hiccups - eventually the reconnection can fail entirely
3. **systemd-networkd vs NetworkManager**: Arch installations may use different network managers - always verify which one is active
4. **The `iw` utility**: Essential tool for diagnosing and configuring WiFi on Linux

## Related Commands Reference

```bash
# Check which network manager is running
systemctl status NetworkManager
systemctl status systemd-networkd

# View network logs
journalctl -u systemd-networkd -f          # Live logs
journalctl -u systemd-networkd -b -1       # Previous boot
journalctl -u systemd-networkd --since "23:00"  # Time range

# WiFi interface management
ip link show                                # List all network interfaces
iw dev                                      # Show wireless devices
iw dev wlan0 link                          # Show connection info
iw dev wlan0 get power_save                # Check power save status
iw dev wlan0 set power_save off            # Disable power save

# Check RF kill status
rfkill list                                # Show all wireless devices
rfkill unblock all                         # Unblock if needed
```

## Prevention
For future Arch installations, consider disabling WiFi power save immediately after setup if stable connectivity is critical for your workflow.

## Additional Notes

### Understanding WiFi Power Management
WiFi power management is a battery-saving feature where the wireless card periodically enters low-power sleep mode. While this extends battery life, it can cause:
- Brief connection drops (1-2 seconds)
- Complete connection failures requiring restart
- Interrupted SSH sessions, video calls, and downloads

### When to Disable Power Save
- **Always disable**: Desktop systems, docked laptops, or when plugged in
- **Highly recommended**: DevOps work, remote sessions, video conferencing
- **Consider keeping**: Casual browsing on battery when reliability isn't critical

### Battery Impact
Disabling WiFi power save typically adds only 5-10 minutes of battery drain over several hours of use - a worthwhile trade-off for stability in most professional contexts.

---

**Date**: January 19, 2026  
**Immediate Fix**: System restart  
**Permanent Fix**: Disabled WiFi power management  
**Duration**: ~20 minutes troubleshooting  
**Status**: ✅ Resolved