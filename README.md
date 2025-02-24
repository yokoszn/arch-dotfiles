
# Hyprland 

---
This covers the major keybindings and usage instructions for **my** Hyprland setup. You’ll see references to programs or scripts that **you** can customize to **your preferences.**

**Note**: Make sure you define your favorite: `$editor` (Obsidian), `$filemanager` (Nemo), `$terminal`, and `$browser` in dotfiles/Envs.conf

## Console / Terminal Hotkeys
- **SUPER + Return**: Opens a new terminal showing `fastfetch` with a specific config.  
- **SUPER + Shift + Return**: Opens a floating terminal at 50% size.

## Basic Window Navigation
- **SUPER + Q**: Closes (kills) the currently active window.  
- **SUPER + Shift + V**: Pins a window to stay on top.  
- **SUPER + Shift + C**: Centers the active window on screen.  
- **Alt + Return**: Toggles fullscreen on the active window.  
- **SUPER + Tab**: Toggles whether the active window floats or tiles.  
- **SUPER + R**: Toggles the window’s split orientation (horizontal/vertical).

## Scratchpad
- These lines define **scratchpad-like workspaces** named `overview` and `running`.  
- **SUPER + ~** toggles the special workspace named `overview`. 
- Typically, `code:49` maps to the key above Tab (`~ / \`` on some keyboards), but confirm your own keycodes with `xev` or similar tools.

## Scrolling Between Workspaces
- **SUPER + Scroll**: Cycles through available workspaces (scroll up or down).  
- **SUPER + Left Mouse Button** (hold & drag): Moves the active window.  
- **SUPER + Right Mouse Button** (hold & drag): Resizes the active window.
- **SUPER + Shift + Arrow**: Moves the window in that direction.  
- **SUPER + Arrow**: Resizes the active window (width/height changes).

## Player & Notification Control
- **SUPER + I**: Toggles play/pause in your media player via `playerctl`.  
- **SUPER + X**: Toggles notifications through `swaync-client`.
  
- For more, check the official [**Hyprland Wiki**](https://wiki.hyprland.org/)

# Miscellanous 
```
Shell : Fish in kitty
nvim : (nvchad - gruvbox)
```
## Laptop Brightness & Lid Switch

```
bindel = ,XF86MonBrightnessUp, exec, brightnessctl s 10%+
bindel = ,XF86MonBrightnessDown, exec, brightnessctl s 10%-
bindl=,switch:off:Lid Switch,exec,hyprctl keyword monitor "eDP-1, preferred, auto, auto"
bindl=,switch:on:Lid Switch,exec,hyprctl keyword monitor "eDP-1, disable"
```

- **Laptop Brightness**: Pressing the dedicated brightness keys (XF86MonBrightnessUp/Down) calls `brightnessctl`. Adjust the percentage values to your preference.  
- **Lid Switch**: Disables or re-enables `eDP-1` monitor.  --- f you’re on a **desktop**, you can remove these.
---
## To Do
- [ ] yubi-key for MFA with decryption with encrypted boot partition.
- [ ] headless encrypted disk setup guide - bootloader on seperate drive, e.g. USB or microSD card
- [ ] Screenshots

