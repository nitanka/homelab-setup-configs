# Linux Absolute No-Sleep Configuration

This guide provides a comprehensive setup to completely disable sleep, hibernation, and automatic suspension features on a Linux environment running the GNOME Desktop Manager (GDM). This is particularly useful for setting up **home servers, media hubs, or remote builds** that must remain reachable 24/7.

## Overview of What This Script Does
1. **Kernel/Boot Level:** Modifies the bootloader behavior (via GRUB editing).
2. **System Level:** Masks core `systemd` sleep and hibernation hooks.
3. **Login Screen Level:** Disables inactivity sleep for the GDM interface.
4. **User Session Level:** Modifies GNOME settings to prevent idle sleep.

---

## Step-by-Step Implementation

### Step 1: GRUB Configuration
Modify the bootloader settings if necessary to add specific hardware or power parameters.
```bash
sudo nano /etc/default/grub
```
*After editing, update the bootloader configuration to save changes:*
```bash
sudo update-grub
```

### Step 2: Mask Systemd Targets
Completely block all system-level suspension states by linking them to `/dev/null`.
```bash
sudo systemctl mask suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

### Step 3: Configure the GDM Login Screen
Prevent the login screen from falling asleep before a user logs in.
```bash
# Create the directory for GDM dconf adjustments
sudo mkdir -p /etc/dconf/db/gdm.d

# Write the power-saving override configuration
cat <<EOF | sudo tee /etc/dconf/db/gdm.d/01-power
[org/gnome/settings-daemon/plugins/power]
sleep-inactive-ac-timeout=0
sleep-inactive-ac-type='nothing'
EOF

# Apply system database updates
sudo dconf update
```

### Step 4: Configure the Current User Session
Disable idle timeout actions for your active GNOME desktop user session.
```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 0
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
```

---

## Verification

To verify that the system is successfully locked against accidental sleep, run:
```bash
systemctl status suspend.target
```

**Expected Output Snippet:**
```text
● suspend.target
     Loaded: masked (Reason: Unit suspend.target is masked.)
     Active: inactive (dead)
```
If the status shows `masked`, your system will successfully stay awake indefinitely.

---

## Reverting Changes
If you ever need to restore default sleep functionality, run:
```bash
sudo systemctl unmask suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
gsettings reset org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout
gsettings reset org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type
sudo rm /etc/dconf/db/gdm.d/01-power && sudo dconf update
```
