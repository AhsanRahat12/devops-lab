## Fixing Hyprland windowrulev2 Deprecation Errors

### Issue
Hyprland 0.53.2 showed config errors about deprecated `windowrulev2` syntax on lines 323 and 326 of `~/.config/hypr/hyprland.conf`.

### Root Cause
`windowrulev2` is completely deprecated in Hyprland 0.53.2. The new syntax uses `windowrule` with a different format:
- Props use `match:property` format
- Effects are specified differently
- Multiple conditions can be chained or use named rule blocks

### Solution

**Line 323 - Suppress maximize events:**
```conf
# Old (deprecated)
windowrulev2 = suppressevent maximize, class:.*

# New (correct)
windowrule = match:class .*, suppress_event maximize
```

**Line 326 - No focus for specific windows:**
```conf
# Old (deprecated)
windowrulev2 = nofocus, class:^$, title:^$, xwayland:1, floating:1, fullscreen:0, pinned:0

# New (correct)
windowrule = no_focus on, match:class ^$, match:title ^$, match:xwayland 1, match:float 1, match:fullscreen 0, match:pin 0
```

### Commands Used
```bash
# Backup config
cp ~/.config/hypr/hyprland.conf ~/.config/hypr/hyprland.conf.backup

# Fix line 323
sed -i '323c\windowrule = match:class .*, suppress_event maximize' ~/.config/hypr/hyprland.conf

# Fix line 326
sed -i '326c\windowrule = no_focus on, match:class ^$, match:title ^$, match:xwayland 1, match:float 1, match:fullscreen 0, match:pin 0' ~/.config/hypr/hyprland.conf

# Reload config
hyprctl reload
```

### Key Takeaways
- Always check Hyprland wiki when seeing deprecation warnings
- `windowrulev2` → `windowrule` with `match:property` syntax
- Use `hyprctl version` to verify Hyprland version for correct syntax reference
- `hyprctl reload` applies config changes without restarting session
- Reference: https://wiki.hyprland.org/Configuring/Window-Rules/
