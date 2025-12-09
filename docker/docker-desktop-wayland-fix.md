# Docker Desktop Wayland Scaling Fix

**Environment:** Arch Linux with Wayland (Hyprland)  
**Issue:** Blurry Docker Desktop display despite other Electron apps rendering correctly  
**Root Cause:** Dual-process architecture requiring configuration at multiple levels

---

## The Problem

Docker Desktop appeared blurry on Wayland even after other Electron applications (Obsidian, VS Code) were rendering correctly with standard Wayland flags. Simply adding flags to the desktop file, which worked for single-process apps, was insufficient for Docker Desktop.

---

## Why Docker Desktop is Different

### Dual-Process Architecture

Unlike typical Electron applications that run as a single process, Docker Desktop uses two separate components:

**1. Backend Service (`com.docker.backend`)**
- Runs as a systemd user service
- Manages Docker engine and containers
- Starts automatically via systemd
- Handles core Docker functionality

**2. GUI Frontend (`docker-desktop`)**
- Electron-based user interface
- Launched from the desktop file
- Communicates with the backend service
- Displays container status, settings, and controls

**The Challenge:**
- Desktop file flags only affect the GUI process
- The backend service (started by systemd) doesn't inherit those flags
- Both processes must be configured for native Wayland support
- If either process uses XWayland, the display quality suffers

---

## The Solution: Dual Configuration

Both components must be configured separately to enable native Wayland rendering.

### Component 1: Systemd Service Override

Configure the backend service to run with Wayland support:

```bash
# Create override directory
mkdir -p ~/.config/systemd/user/docker-desktop.service.d

# Create override configuration file
cat > ~/.config/systemd/user/docker-desktop.service.d/wayland.conf << 'EOF'
[Service]
Environment="ELECTRON_ENABLE_WAYLAND=1"
Environment="ELECTRON_OZONE_PLATFORM_HINT=wayland"
EOF

# Reload systemd to apply changes
systemctl --user daemon-reload
```

**What this does:**
- Sets Wayland environment variables for the backend service
- Ensures the backend process initializes with Wayland support
- Takes effect when the service starts

### Component 2: Desktop File Configuration

Configure the GUI frontend:

```bash
# Copy system desktop file to user directory for customization
cp /usr/share/applications/docker-desktop.desktop ~/.local/share/applications/

# Edit the desktop file
nano ~/.local/share/applications/docker-desktop.desktop
```

Modify the `Exec=` line to:

```
Exec=env ELECTRON_ENABLE_WAYLAND=1 ELECTRON_OZONE_PLATFORM_HINT=wayland /opt/docker-desktop/bin/docker-desktop --enable-features=UseOzonePlatform --ozone-platform=wayland
```

**What this does:**
- Sets Wayland environment variables for the GUI process
- Enables Ozone platform with native Wayland backend
- Applies when launching from application menu

---

## Installation and Setup

### Fresh Installation Steps

If starting from scratch or troubleshooting:

**1. Install Docker Desktop via AUR**
```bash
paru -S docker-desktop
```

**Important:** When prompted for QEMU options, select `qemu-desktop` (not qemu-base) as it includes GUI support.

**2. Apply Both Configurations**

Follow the steps in "The Solution" section above to configure both the systemd service and desktop file.

**3. Start Docker Desktop**
```bash
systemctl --user start docker-desktop
```

Or launch from your application menu.

---

## Verification

After applying both configurations:

1. Launch Docker Desktop from your application launcher
2. The interface should render crisp and clear
3. Text should be sharp and readable
4. UI elements should scale properly

---

## Configuration Reference

### Systemd Service Override
**Location:** `~/.config/systemd/user/docker-desktop.service.d/wayland.conf`

```ini
[Service]
Environment="ELECTRON_ENABLE_WAYLAND=1"
Environment="ELECTRON_OZONE_PLATFORM_HINT=wayland"
```

### Desktop File
**Location:** `~/.local/share/applications/docker-desktop.desktop`

```ini
[Desktop Entry]
Exec=env ELECTRON_ENABLE_WAYLAND=1 ELECTRON_OZONE_PLATFORM_HINT=wayland /opt/docker-desktop/bin/docker-desktop --enable-features=UseOzonePlatform --ozone-platform=wayland
```

---

## Troubleshooting

### Display Still Blurry

**1. Verify both configuration files exist:**
```bash
cat ~/.config/systemd/user/docker-desktop.service.d/wayland.conf
cat ~/.local/share/applications/docker-desktop.desktop
```

**2. Reload systemd configuration:**
```bash
systemctl --user daemon-reload
```

**3. Completely restart Docker Desktop:**
```bash
systemctl --user stop docker-desktop
systemctl --user start docker-desktop
```

**4. Check for errors:**
```bash
journalctl --user -u docker-desktop -f
```

**5. Test manual launch:**
```bash
/opt/docker-desktop/bin/docker-desktop --enable-features=UseOzonePlatform --ozone-platform=wayland
```

### Service Not Starting

**Check service status:**
```bash
systemctl --user status docker-desktop
```

**View detailed logs:**
```bash
journalctl --user -u docker-desktop -n 50
```

---

## Key Takeaways

**Why this fix is necessary:**
- Docker Desktop's dual-process architecture requires configuration at multiple levels
- Systemd service and desktop file must both specify Wayland support
- Single-point configuration (desktop file only) is insufficient

**Configuration locations:**
- Backend service: `~/.config/systemd/user/docker-desktop.service.d/`
- GUI frontend: `~/.local/share/applications/docker-desktop.desktop`

**Important principles:**
- User-level overrides take precedence over system files
- Systemd requires `daemon-reload` after configuration changes
- Both processes must use native Wayland to avoid blur

---

## Related Notes

This dual-process configuration approach differs from single-process Electron apps. For understanding the underlying Wayland scaling issues that affect all Electron applications, see the general Wayland scaling documentation.

**Result:** Docker Desktop now renders with crisp, properly-scaled display on Wayland.
