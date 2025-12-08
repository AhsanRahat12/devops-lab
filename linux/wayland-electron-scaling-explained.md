# Wayland Electron App Scaling Issues - Technical Deep Dive

**Environment:** Arch Linux with Wayland (Hyprland)  
**Affected Apps:** Electron-based applications (Obsidian, VS Code, Discord, Slack, etc.)  
**Issue:** Blurry text and UI elements on high-DPI displays

---

## The Problem

Electron applications appear blurry on Wayland when using fractional scaling (125%, 150%, 175%). This affects popular tools like Obsidian, VS Code, and Discord, making them difficult to use despite otherwise functioning correctly.

---

## Root Cause: Wayland vs X11 Rendering

### The Fundamental Issue

Electron apps are built on Chromium, which historically used X11 (the older Linux display server protocol). When running on Wayland (the modern replacement), these apps default to running through **XWayland** - a compatibility layer that translates X11 calls to Wayland.

This compatibility layer works, but it wasn't designed for modern high-DPI displays with fractional scaling.

### What Causes the Blurriness?

**1. Fractional Scaling Mismatch**
- Modern displays use fractional scaling (125%, 150%, 175%) for optimal readability
- XWayland was designed for integer scaling only (100%, 200%, etc.)
- When an app renders at one scale and the compositor scales it differently, you get blurry text and UI

**2. Double Scaling Problem**
- App renders at 100% (assuming it's on a standard display)
- Wayland compositor then scales the entire window by 1.5x (for a 150% display setting)
- This post-render scaling causes blur, similar to enlarging a low-resolution image

**3. Buffer Mishandling**
- XWayland creates rendering buffers at the wrong resolution
- Wayland has to interpolate/scale these buffers to fit the display
- Interpolation = blur

### Technical Flow Comparison

**Without proper configuration (blurry):**
```
Electron App → Assumes X11 → Renders at 100% → 
XWayland translates → Wayland scales 1.5x → BLUR
```

**With proper configuration (crisp):**
```
Electron App → Native Wayland → Renders at correct scale → 
Direct to Wayland → No additional scaling → CRISP
```

---

## The Solution: Configuration Flags

### Ozone Platform Flags: `--enable-features=UseOzonePlatform --ozone-platform=wayland`

**What it does:**
- Tells Chromium/Electron to use the Ozone abstraction layer
- Ozone is Chromium's platform abstraction supporting multiple backends (X11, Wayland, etc.)
- `--ozone-platform=wayland` specifically selects native Wayland rendering
- Bypasses XWayland entirely - the app communicates directly with Wayland

**Result:** The app becomes a native Wayland client, understanding Wayland's scaling protocol correctly and rendering at the appropriate scale.

### Environment Variables: `ELECTRON_ENABLE_WAYLAND=1` and `ELECTRON_OZONE_PLATFORM_HINT=wayland`

**What they do:**
- Set Electron-specific hints before the app launches
- Ensure Electron initializes with Wayland support enabled
- `ELECTRON_OZONE_PLATFORM_HINT` guides Electron to prefer Wayland over X11

**Result:** Electron knows from startup to use native Wayland rendering paths.

---

## Display Server Evolution Context

### X11 (Legacy)
- Simple integer scaling only
- Apps didn't need to be "scale-aware"
- Works but looks poor on high-DPI displays

### Wayland (Modern)
- Full fractional scaling support
- Apps must be explicitly Wayland-aware
- Better for modern high-DPI displays but requires application support

### XWayland (Compatibility Layer)
- Allows X11 apps to run on Wayland
- Lowest common denominator approach
- Works but with quality compromises (like blur on high-DPI displays)

---

## Implementation Methods

### Method 1: Desktop File Configuration (GUI Launches)

Edit the application's `.desktop` file in `~/.local/share/applications/`:

```ini
[Desktop Entry]
Name=Obsidian
Exec=obsidian --enable-features=UseOzonePlatform --ozone-platform=wayland
Terminal=false
Type=Application
Icon=obsidian
```

This configuration applies when launching from GUI application menus (wofi, rofi, etc.).

**Limitation:** Must be configured separately for each application.

### Method 2: Shell Alias (Terminal Launches)

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias obsidian='obsidian --enable-features=UseOzonePlatform --ozone-platform=wayland'
```

This ensures terminal launches also use the correct flags.

**Limitation:** Must be configured separately for each application.

### Method 3: Global Environment Variables (Recommended)

Add to shell configuration (`~/.bashrc` or `~/.zshrc`):

```bash
export ELECTRON_ENABLE_WAYLAND=1
export ELECTRON_OZONE_PLATFORM_HINT=wayland
```

**Why I chose this method:**
- Set once and it applies to ALL Electron apps system-wide
- No need to configure individual `.desktop` files for each app
- No need to create aliases for terminal launches
- Works consistently regardless of how apps are launched
- Simpler to maintain - one configuration point instead of many

This is the approach I use on my system, as it provides the most consistent experience across all Electron applications without repeated configuration.

---

## Key Takeaways

**Why blurriness occurs:**
1. Electron apps default to X11 compatibility mode (XWayland)
2. XWayland doesn't understand fractional scaling properly
3. Post-render scaling by the compositor causes interpolation blur

**How the fix works:**
1. Enable native Wayland rendering (bypass XWayland)
2. Apps automatically detect and use the correct scale factor
3. Pre-render scaling eliminates interpolation blur

**Why this matters for DevOps:**
- Understanding display server architecture helps troubleshoot GUI apps in containers
- Demonstrates ability to debug at the system architecture level
- Essential knowledge for running desktop applications in modern Linux environments
- Applicable across many Electron-based development tools

---

## Related Issues

This same approach solves scaling issues for:
- VS Code / VSCodium
- Obsidian
- Discord
- Slack
- Any other Electron-based application

The underlying principle remains the same: enable native Wayland support so apps can properly handle display scaling.
