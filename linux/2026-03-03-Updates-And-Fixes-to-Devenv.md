# Dotfiles Setup Fixes — Summary

A record of issues encountered and fixes made while bootstrapping the chezmoi dotfiles system across Arch Linux and two Raspberry Pis (luffy and zoro).

---

## 1. Hardcoded PATH in `dot_bashrc`

**Problem:** The `.bashrc` had a hardcoded path `/home/rahat/.rd/bin` which only worked on the Arch machine where the username is `rahat`. On the Pis where the username is `pi`, this path didn't exist and caused issues.

**Fix:** Replaced all hardcoded home paths with `$HOME` so they resolve correctly for any user on any machine:

```bash
# Before
export PATH="/home/rahat/.rd/bin:$PATH"

# After
export PATH="$HOME/.rd/bin:$PATH"
export PATH="$HOME/bin:$PATH"
export PATH="$HOME/.local/bin:$PATH"
```

The last two lines also ensure the chezmoi and mise binaries are on PATH after install.

---

## 2. Starship Not Auto-Installing on New Machines

**Problem:** When bootstrapping a new Pi, starship was not installed automatically. This meant the shell prompt looked like the default bash prompt instead of the custom starship theme.

**Root cause:** `starship` was missing from the mise config file, so mise had no instruction to install it.

**Fix:** Added starship to `~/.config/mise/config.toml`:

```toml
[tools]
bat = "latest"
chezmoi = "latest"
python = "latest"
starship = "latest"   # ← added this
```

---

## 3. Starship Config Not Applying on Pis (Broken Symlink)

**Problem:** The `starship.toml` was stored in chezmoi as a symlink (`symlink_starship.toml`) pointing to an Arch-specific path that didn't exist on the Pis. This meant the starship config was never applied and the prompt didn't look the same across machines.

**Fix:** Removed the symlink and added the actual `starship.toml` file directly into the chezmoi source directory so chezmoi copies it properly to all machines.

---

## 4. Mise Trust Not Fully Automated

**Problem:** On fresh Pi installs, mise would throw errors about untrusted config files:

```
mise ERROR Config files in ~/.local/share/chezmoi/dot_config/mise/config.toml are not trusted.
```

The existing trust script only trusted the applied config at `~/.config/mise/config.toml` but missed the chezmoi **source** config at `~/.local/share/chezmoi/dot_config/mise/config.toml`. Mise sees both and complains about the second.

**Fix:** Added a second trust line to the chezmoi script:

```bash
#!/bin/bash
# packages hash: {{ include "dot_config/mise/config.toml" | sha256sum }}
$HOME/.local/bin/mise trust $HOME/.config/mise/config.toml
$HOME/.local/bin/mise trust $HOME/.local/share/chezmoi/dot_config/mise/config.toml
$HOME/.local/bin/mise install
```

This ensures both the applied dotfile and the chezmoi source file are trusted, and all tools are installed automatically on every new machine.

Please note the the normal command would be :
```
mise <command> <file path>
## but in our script we use the full file path as our new / minimal environment might not have the full $PATH in it.

```

---

## 5. Hostname in Starship Prompt

**Problem:** The starship prompt only showed `$username` (always `pi`) which made it impossible to tell the two Pis apart when SSHed in.

**Fix:** Replaced `$username` with `$hostname` in the starship format string and added a `[hostname]` config block:

```toml
[hostname]
ssh_only = false
style = "bg:surface0 fg:text"
format = '[ $hostname ]($style)'
disabled = false
```

Now the prompt shows the machine hostname (`luffy` or `zoro`) making it easy to know which Pi you're on.

---

## Bootstrap Command

To set up a new machine from scratch:

```bash
sh -c "$(curl -fsLS get.chezmoi.io)" -- -b $HOME/.local/bin init --apply https://github.com/AhsanRahat12/dotfiles
```

This installs chezmoi, clones the dotfiles repo, and applies everything — including trusting mise and installing all tools — in one shot.
