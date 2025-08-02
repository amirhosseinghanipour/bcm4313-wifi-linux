# BCM4313 WiFi Issue

## Overview
This repository tracks the troubleshooting and resolution process for a WiFi connectivity issue on a Dell Inspiron N5040 running Arch Linux with i3wm and X11. The WiFi adapter is a **Broadcom BCM4313 802.11bgn Wireless Network Adapter (rev 01)**. The issue is that the WiFi connection initially works but experiences speed drops and loss of internet connectivity despite remaining connected to the network. I'd like to diagnose and fix the issue in this repo and potentially contribute to a stable driver solution.

## System Details
- **Hardware**: Dell Inspiron N5040
- **WiFi Chipset**: Broadcom BCM4313 802.11bgn Wireless Network Adapter (rev 01)
  - `lspci` output: `12:00.0 Network controller: Broadcom Inc. and subsidiaries BCM4313 802.11bgn Wireless Network Adapter (rev 01)`
- **OS**: Arch Linux
- **Window Manager**: i3wm (X11 environment)
- **Network Management**: `iwd` (iNet Wireless Daemon) with `systemd-networkd`
- **WiFi Interface**: `wlan0`
- **Router**: SSID `v1ll3n`, 2.4 GHz band
- **Symptoms**:
  - WiFi connects successfully but speed drops significantly after some time.
  - Internet access fails despite `wlan0` showing as connected.
  - Issue persists across kernel versions, Linux distributions, and router distances.
  - Only this device on the network experiences the issue.
- **Logs**:
  - `iwd` logs show repeated deauthentications (e.g., `Received Deauthentication event, reason: 1, from_ap: true` and `reason: 4`).
  - `lsmod | grep -E 'wlan|wifi|wireless'` initially returned nothing, indicating no proper WiFi driver was loaded.

## Steps Taken
1. **Initial Diagnosis** (2025-08-02):
   - Ran `lspci | grep -i wireless` to confirm the WiFi chipset as BCM4313.
   - Ran `lsmod | grep -E 'wlan|wifi|wireless'`; no WiFi modules were loaded.
   - Ran `iwconfig` to check the WiFi interface status: `wlan0     IEEE 802.11  ESSID:off/any  Mode: Managed  Access Point: Not-Associated   Tx-Power=19 dBm Retry short limit:7   RTS thr:off   Fragment thr:off Power Management:off`
   - Confirmed network management uses `iwd` and `systemd-networkd` (who even uses `NetworkManager`).
   - Ran `inxi -N` to confirm the driver as `bcma-pci-bridge` a bus driver.
   - Reviewed `journalctl -u iwd.service` logs repeated showed connection attempts and deauthentications (reasons 1 JN4).
   - Disabling IPv6 (`ipv6.disable=1`) had no effect.
   - Disabling power-saving modes had no effect.
   - Testing with different kernels and Linux distributions yielded the same issue.
   - Only this device is affected, regardless of distance from the router.

2. **Driver Troubleshooting Attempt - brcmsmac** (2025-08-02):
   - I realized the `bcma-pci-bridge` driver wasn’t cutting it for WiFi, so I decided to try the open-source `brcmsmac` driver (part of `brcm80211`) to see if it could get my WiFi working properly.
   - Installed the `linux-firmware` package to make sure the necessary firmware was available:
     ```bash
     sudo pacman -S linux-firmware
     ```
   - Loaded the `brcmsmac` module:
     ```bash
     sudo modprobe brcmsmac
     ```
   - Checked if the module loaded correctly with `lsmod | grep brcmsmac`, which showed:
     ```
     brcmsmac              724992  0
     brcmutil               20480  1 brcmsmac
     cordic                 12288  2 b43,brcmsmac
     mac80211             1650688  2 b43,brcmsmac
     cfg80211             1400832  3 b43,mac80211,brcmsmac
     bcma                   86016  2 b43,brcmsmac
     rfkill                 45056  7 bluetooth,dell_laptop,brcmsmac,cfg80211
     ```
   - To avoid conflicts with `brcmsmac`, I blacklisted the `bcma` driver:
     ```bash
     echo "blacklist bcma" | sudo tee /etc/modprobe.d/blacklist-bcma.conf
     ```
   - Rebooted to apply the changes:
     ```bash
     sudo reboot
     ```
   - **Result**: After the reboot, I was gutted to find that my WiFi interface (`wlan0`) had completely disappeared. It seems blacklisting `bcma` broke things, likely because `brcmsmac` couldn’t properly initialize my BCM4313 chipset.

3. **Driver Troubleshooting Attempt - broadcom-wl** (2025-08-02):
   - Since `brcmsmac` didn’t work out, I switched to the proprietary `broadcom-wl` driver from the AUR, hoping it would save the day:
     ```bash
     yay -S broadcom-wl
     ```
   - Made sure conflicting drivers (`brcmsmac` and `bcma`) were blacklisted:
     ```bash
     echo -e "blacklist brcmsmac\nblacklist bcma" | sudo tee /etc/modprobe.d/blacklist-broadcom.conf
     ```
   - Loaded the `wl` module:
     ```bash
     sudo modprobe wl
     ```
   - Connected to my `v1ll3n` SSID using `iwctl`:
     ```bash
     iwctl
     device list
     station wlan0 scan
     station wlan0 get-networks
     station wlan0 connect v1ll3n
     ```
   - **Result**: Success! The `wl` driver worked like a charm. My WiFi connection is now stable, with no speed drops or disconnections when my laptop is near the router.

## Resolution
- The proprietary `broadcom-wl` driver fixed my WiFi issue.
- The `brcmsmac` driver was a no-go—blacklisting `bcma` made my WiFi interface vanish, the BCM4313 (rev 01) isn’t well-supported by `brcmsmac`.
- With the `wl` driver, my connection holds strong.

I hope it helps others facing the same headache and maybe even sparks some improvements for Broadcom WiFi support in Linux. Feel free to share feedback.

## Notes
- The `wl` driver is proprietary and might not play nice with future kernel updates. I’ll keep an eye on AUR updates and might consider a USB WiFi adapter with better open-source support (like Atheros or Realtek) for the long haul.
- The `brcmsmac` failure after blacklisting `bcma` shows that the BCM4313 (rev 01) probably needs specific firmware or patches that aren’t in the current Linux kernel.
