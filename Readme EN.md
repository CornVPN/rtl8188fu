# Installing RTL8188FU Driver on Bazzite OS (Fedora Silverblue/Atomic)

A guide on how to manually compile and set up persistent persistent auto-loading for the Realtek RTL8188FU (USB) Wi-Fi adapter on immutable/atomic Linux environments where standard DKMS is not supported.

## 🛠️ Step 1. Prepare System Dependencies

Ensure that kernel headers (`kernel-devel`) are layered on your system via rpm-ostree:
```bash
sudo rpm-ostree install kernel-devel
```
*If it says "already requested", skip this step. If it installs fresh, you **must reboot** your PC before proceeding.*

## 🔨 Step 2. Compile the Module

1. Clone the driver repository (e.g., from `kelebek33`) and navigate to the directory:
```bash
cd ~/rtl8188fu
```
2. Clear any old build cache and compile the kernel module manually:
```bash
make clean
make
```

## 🚫 Step 3. Blacklist Conflicting Kernel Driver

The built-in kernel module `rtl8xxxu` natively claims this chip but causes hard crashes. Permanently blacklist it via the bootloader arguments:
```bash
sudo rpm-ostree kargs --append="modprobe.blacklist=rtl8xxxu"
```
**You must reboot your PC now** to apply the new kernel boot arguments!

## ⚙️ Step 4. Tweak NetworkManager for Stability

To prevent endless scanning loops and handshake disconnections, disable randomized MAC addresses and aggressive Wi-Fi powersave:

```bash
sudo nano /etc/NetworkManager/conf.d/99-rtl-fix.conf
```
Paste the following configuration:
```ini
[device]
wifi.scan-rand-mac-address=no

[connection]
wifi.cloned-mac-address=preserve
wifi.powersave=2
```

## 🚀 Step 5. Setup Persistent Auto-load (Bazzite-specific Workaround)

Standard `systemd` services loading modules out of home directories get blocked by SELinux on Fedora Atomic systems. To bypass this, we use the privileged `rc.local` boot script:

1. Open or create the startup script:
```bash
sudo nano /etc/rc.d/rc.local
```
2. Insert the script below. It includes critical flags to **disable aggressive chip power management** (which causes immediate `scanning -> disconnected` drops) and forces a **20 MHz channel width** for bulletproof stability:
```bash
#!/bin/bash
/usr/sbin/modprobe cfg80211
/usr/sbin/insmod /home/icuxu/rtl8188fu/rtl8188fu.ko rtw_ht_enable=0 rtw_ips_mode=0 rtw_power_mgnt=0
exit 0
```
3. Make the script executable:
```bash
sudo chmod +x /etc/rc.d/rc.local
```

You can test it right away by running `sudo /etc/rc.d/rc.local` or simply restart your system. Your Wi-Fi adapter will now initialize automatically on every boot!
