# Kitty Terminal + Less Search Issue

## Problem
Unable to type search terms in `less` after pressing `/` in Kitty terminal.

## Cause
Kitty uses `xterm-kitty` terminfo which has incompatibility with how `less` handles input for search commands. This is a known issue with Kitty's keyboard protocol.

## Solution

### Option 1: Permanent Fix (Recommended)
Add to `~/.bashrc`:
```bash
export TERM=xterm-256color
```

Then reload:
```bash
source ~/.bashrc
```

### Option 2: Fix Only for Less
Create an alias in `~/.bashrc`:
```bash
alias less="TERM=xterm-256color less"
```

### Option 3: One-time Fix
```bash
TERM=xterm-256color less filename
```

## Result
After applying the fix, search functionality (`/` command) works properly in less.

## Alternative Solutions
- Use `grep` for quick searches: `kubectl run nginx --help | grep -i "search term"`
- Use `vim` instead: `kubectl run nginx --help | vim -`