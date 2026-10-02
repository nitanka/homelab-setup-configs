# Proxmox VE 2-Node Cluster Setup & Administration Guide

This document details the complete end-to-end installation of Proxmox VE on Debian 13 (Trixie), cluster creation, service auto-start configuration, and post-installation UI tweaks (disabling the subscription banner).

## Environment Overview

* **Node 1 :** `192.168.4.111` (VLAN 4)

* **Node 2 :** `192.168.3.111` (VLAN 3)

* **Switch:** Managed Switch

## Section 1: Proxmox VE Installation on Debian

Follow these steps on each node to install Proxmox VE on top of a standard Debian installation.

### 1. Configure `/etc/hosts`

Proxmox requires the hostname to resolve directly to the static IP address (not `127.0.0.1` or `127.0.1.1`).

Edit `/etc/hosts`:

```
127.0.0.1       localhost
192.168.4.111   master.local master  # (Use 192.168.3.111 and hostname on Node 2)

::1             localhost ip6-localhost ip6-loopback

```

### 2. Add Proxmox VE Repository & GPG Keys

```
# Add Proxmox No-Subscription repository
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list

# Import Proxmox release key
wget https://enterprise.proxmox.com/proxmox-release-trixie.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg

# Update package repository index
apt update && apt full-upgrade -y

```

### 3. Install Proxmox Kernel & Core Packages

```
# Install Proxmox kernel
apt install -y proxmox-default-kernel

# Reboot into Proxmox kernel
reboot

```

After rebooting:

```
# Install Proxmox VE core packages
apt install -y proxmox-ve postfix open-iscsi

# (Optional) Remove default Debian kernel
apt remove -y linux-image-amd64 'linux-image-6.1*'
update-grub

```

## Section 2: Cluster Creation & Joining

### 1. Initialize Cluster on Node 1 (`192.168.4.111`)

Run on **Node 1**:

```
# Create cluster referencing Node 1 management IP
pvecm create pve-cluster --link0 192.168.4.111

# Verify cluster status
pvecm status

```

### 2. Join Node 2 (`192.168.3.111`)

Run on **Node 2**:

```
# Join Node 2 to Node 1 across subnets
pvecm add 192.168.4.111 --link0 192.168.3.111

```

1. Confirm SSH fingerprint by typing `yes`.

2. Enter the `root` password for **Node 1**.

### 3. Verify Cluster Membership

From either node, execute:

```
pvecm nodes

```

## Section 3: Enable Auto-Start at Boot

To ensure all Proxmox daemons, Corosync, and networking start automatically on boot, execute the following on **both nodes**:

```
# Enable Proxmox cluster, API, and HA CRM/LRM services
systemctl enable corosync pve-cluster pvedaemon pveproxy pve-ha-crm pve-ha-lrm

# Ensure network wait-online service is enabled to prevent race conditions
systemctl enable networking systemd-networkd-wait-online.service

```

## Section 4: Disable the "No Subscription" Web UI Banner

By default, logging into the Proxmox Web UI pops up a warning dialogue stating *"You do not have a valid subscription for this server"*. You can disable this banner cleanly via CLI or post-installation script.

### Option A: Quick CLI Mod (Proxmox VE 7 / 8 / 9)

Run the following command on **both nodes** to patch the Web UI JavaScript file and restart `pveproxy`:

```
# Backup original file
cp /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js.bak

# Patch out the subscription check dialogue
sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js

# Restart the Proxmox Web Proxy service
systemctl restart pveproxy.service

```

> **Note:** Clear your browser cache or perform a hard refresh (`Ctrl + F5` / `Cmd + Shift + R`) when opening `https://<NODE_IP>:8006` to load the modified script without the banner.

### Option B: Using Proxmox Helper Utility Script

Alternatively, you can run the community-maintained post-install tool:

```
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/misc/post-pve-install.sh)"

```

Select **Yes** when prompted to *Disable "No Subscription" Nag*.