# Hardware & Network Link Flap Troubleshooting

This document records the diagnosis and resolution for persistent daily physical link drops, carrier `linkdown` events, and negotiation failures on Realtek network adapters (`r8169`) connected to managed switches.

---

## Problem Summary

* **Symptoms:** Intermittent loss of SSH/Ping, physical network LEDs blinking orange without green activity, `ip route` displaying `linkdown`, and loss of `vmbr0` IP assignments.
* **Root Cause:**
  1. **PCIe ASPM Power Savings:** Kernel-level PCIe Active State Power Management putting the network interface into unrecoverable deep sleep.
  2. **Energy Efficient Ethernet (EEE):** Auto-negotiation failure loops between Realtek PHY and TP-Link SG3210 switch ports.
  3. **Bridge Route Drop:** Linux bridge (`vmbr0`) dropping static IPv4 routing when the underlying physical interface (`enp0s31f6` / `eth0`) loses carrier state.

---

## Applied Solutions

### 1. Disable Kernel-Level ASPM and EEE (`/etc/modprobe.d/r8169.conf`)

Applied on **both nodes** to force the `r8169` driver to keep the PHY powered up continuously:

```bash
# 1. Create module options file
echo "options r8169 aspm=0 eee_enable=0" > /etc/modprobe.d/r8169.conf

# 2. Update initramfs image across all installed kernels
update-initramfs -u -k all
```

---

### 2. Persistent `ethtool` Directives (`/etc/network/interfaces`)

Added `post-up` directives inside the network interfaces file to enforce EEE disablement upon link initialization:

```text
auto lo
iface lo inet loopback

iface enp0s31f6 inet manual
    post-up ethtool --set-eee enp0s31f6 eee off || true

auto vmbr0
iface vmbr0 inet static
    address 192.168.4.111/24
    gateway 192.168.4.1
    bridge-ports enp0s31f6
    bridge-stp off
    bridge-fd 0
```

---

### 3. TP-Link SG3210 Switch Configuration Adjustments

To prevent infinite auto-negotiation loops on managed switch ports:

1. **Port Speed:** Kept set to **Auto** (forcing 1000MF without auto-negotiation breaks IEEE 802.3ab Gigabit standards).
2. **EEE / Green Ethernet:** Disabled globally and per-port under **System $\rightarrow$ Green Ethernet** and **L2 Features $\rightarrow$ Switch Port**.
3. **Flow Control:** Disabled on ports assigned to cluster nodes.

---

## Emergency Recovery Commands

If a link flap occurs and `vmbr0` loses its IP route without a physical reboot:

```bash
# Force-restart network bridge and physical adapter
ifdown vmbr0 && ifup vmbr0

# Disable EEE dynamically
ethtool --set-eee enp0s31f6 eee off

# Manually reinstate gateway route if missing
ip route add default via 192.168.4.1 dev vmbr0
```