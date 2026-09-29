## Custom GRUB Boot & Power Management Configuration

This configuration modifies the GRUB bootloader to ensure the system boots automatically into a specific kernel entry while disabling display sleeping and aggressive PCIe power management.

### Configuration Goals
1. **Set Boot Entry 1 as Default:** Automatically selects the second option in the GRUB boot menu (useful when running specific older kernels or dual-boot setups where index `0` is not desired).
2. **`consoleblank=0`:** Disables the Linux terminal virtual console from turning off/blanking out after periods of inactivity.
3. **`pcie_aspm=off`:** Disables Active State Power Management for PCIe devices. This resolves stability issues, PCIe bus errors, or disconnects with certain NVMe drives and network cards at the cost of slightly higher power consumption.

---

### Implementation Instructions

1. **Open the GRUB configuration file:**
   ```bash
   sudo nano /etc/default/grub
   ```

2. **Modify the following lines to match this exact configuration:**
   ```text
   GRUB_DEFAULT=1
   GRUB_CMDLINE_LINUX_DEFAULT="quiet consoleblank=0 pcie_aspm=off"
   ```
   *(Note: Ensure `GRUB_DEFAULT=1` is set. Linux indexes boot options starting from `0`, so `1` represents the second item in the list).*

3. **Save and exit the editor** (In `nano`, press `Ctrl+O`, `Enter`, then `Ctrl+X`).

4. **Regenerate the bootloader configuration to apply changes:**
   * On **Ubuntu / Debian**:
     ```bash
     sudo update-grub
     ```
   * On **RHEL / CentOS / Fedora**:
     ```bash
     sudo grub2-mkconfig -o /boot/grub2/grub.cfg
     ```

5. **Reboot the system** to initialize the new kernel arguments:
   ```bash
   sudo reboot
   ```

---

### Verification
After the system reboots, verify that your new kernel command-line parameters were successfully loaded by running:
```bash
cat /proc/cmdline
```
**Expected output includes:** `... quiet consoleblank=0 pcie_aspm=off`

