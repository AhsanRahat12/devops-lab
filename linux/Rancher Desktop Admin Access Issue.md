# Rancher Desktop Administrative Access on Hyprland

## Initial Problem
Rancher Desktop's GUI had scaling issues on Hyprland - the administrative access window appeared small and only certain sections were visible, making it impossible to properly interact with the interface.

## Solution Approach
Decided to use the terminal approach instead of the GUI to enable administrative access using:
```bash
rdctl set --application.admin-access=true
```

## Secondary Problem Encountered
When attempting to enable administrative access via terminal, encountered polkit authentication errors. The polkit password prompt failed to appear, preventing the configuration from being applied.

## Root Cause
Hyprland is a minimal window manager and doesn't automatically start a polkit authentication agent. The polkit daemon (`polkitd`) was running, but without an authentication agent, there's no way to show the password prompt when applications like Rancher Desktop need elevated privileges.

## Diagnosis
Checked running processes and configuration:
```bash
# Check running polkit processes
pgrep -a polkit
# Output: Only polkitd was running

# Check installed polkit packages
pacman -Qs polkit
# Output: polkit-kde-agent was installed but not running

# Check Hyprland config for polkit
grep -i polkit ~/.config/hypr/hyprland.conf
# Output: No polkit agent configured
```

## Complete Solution

### Step 1: Configure Polkit Authentication Agent

Add the KDE polkit agent to Hyprland's startup configuration:

1. Edit `~/.config/hypr/hyprland.conf` and add:
```
   exec-once = /usr/lib/polkit-kde-authentication-agent-1
```

2. Reload Hyprland:
```bash
   hyprctl reload
```
   Or log out and log back in.

3. Verify the agent is running:
```bash
   pgrep -a polkit
```
   You should see both `polkitd` and `polkit-kde-authentication-agent-1`.

### Step 2: Enable Administrative Access via Terminal

With polkit working, enable administrative access:

1. Check if Rancher Desktop is running:
```bash
   rdctl list-settings
```

2. If not running, start it:
```bash
   rdctl start
```
   Note: You may see GTK/Wayland errors - these are harmless.

3. Enable Administrative Access:
```bash
   rdctl set --application.admin-access=true
```
   You'll be prompted for your password via a normal-sized polkit authentication window.

4. Verify the setting was applied:
```bash
   rdctl list-settings | grep -i admin
```
   Should show `"adminAccess": true`.

5. Restart Rancher Desktop for changes to take effect:
```bash
   rdctl shutdown
   rdctl start
```

6. Verify Kubernetes service networking:
```bash
   kubectl get services
```
   External-IPs should now be in the same subnet as your host machine.

## Result
- GUI scaling issues bypassed by using terminal commands
- Polkit authentication agent now properly configured
- Administrative access enabled and persists across restarts
- Kubernetes services get proper network bridging with host machine
- Services are accessible with correct IP addressing

## Key Takeaways
1. **GUI Issues on Hyprland**: Electron-based apps like Rancher Desktop often have scaling problems on Wayland/Hyprland. Using the CLI (`rdctl`) is a reliable workaround.

2. **Polkit Requirement**: Hyprland requires manual configuration of a polkit authentication agent for password prompts. Common options include:
   - `polkit-kde-agent` (KDE-based)
   - `polkit-gnome` (GNOME-based)
   - `lxqt-policykit` (lightweight Qt-based)

3. **Terminal Advantage**: The terminal approach (`rdctl set --application.admin-access=true`) triggered a properly-sized polkit password prompt instead of dealing with the scaled-down GUI dialog.
