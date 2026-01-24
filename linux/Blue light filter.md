# Setting Up Blue Light Filter on Arch Linux + Hyprland

**Goal:** Automatic blue light filtering like Night Shift on phones to protect eyes during late-night study sessions

## Step 1: Exploring Options
Initially looked at **hyprsunset** (Hyprland-specific), but discovered it requires manual temperature adjustment - not automatic based on time of day.

## Step 2: Removing hyprsunset
```bash
paru -R hyprsunset
```

## Step 3: Installing gammastep
```bash
sudo pacman -S gammastep
```
Gammastep is a Wayland-compatible fork of redshift that auto-adjusts color temperature based on your location and time of day.

## Step 4: Testing Default Settings
```bash
gammastep -l 49.8:-99.9  # Manitoba coordinates
```
Checked defaults with verbose mode:
```bash
gammastep -v -l 49.8:-99.9
```
**Default values:**
- Day: 6500K (neutral white)
- Night: 4500K (gentle warm)

## Step 5: Finding the Right Temperature
Tested warmer settings for better eye protection during 9pm-3am sessions:
```bash
gammastep -l 49.8:-99.9 -t 6500:3500
```
**Final choice: 3500K at night** - warm enough for serious eye protection without being too orange.

## Step 6: Making it Permanent
Added to `~/.config/hypr/hyprland.conf`:
```bash
exec-once = gammastep -l 49.8:-99.9 -t 6500:3500
```

**Note:** `exec-once` only runs on login/restart, not on config reload!

## Step 7: Manual Start (Before Restart)
To use immediately without restarting:
```bash
gammastep -l 49.8:-99.9 -t 6500:3500 &
```

## Key Differences: Blue Light Filter vs Brightness
- **Blue light filter** (gammastep): Changes color temperature - screen stays bright but looks warmer/orange
- **Brightness control** (brightnessctl): Actually dims the backlight intensity

## Result
Automatic eye protection that:
- Transitions smoothly throughout the day
- Applies 3500K warm tint at night
- Requires zero manual intervention after setup
- Perfect for extended late-night coding/learning sessions

---

## Temperature Reference
- 6500K = Neutral daylight
- 4500K = Gentle warm (default)
- 3500K = Noticeable warm (my setting)
- 3000K = Very warm
- 2700K = Extremely warm/orange