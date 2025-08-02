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
   - Ran `lsmod | grep -E 'wlan|wifi|wireless'`; no WiFi modules were loaded, suggesting an issue with the driver.
   - Ran `iwconfig` to check the WiFi interface status: `wlan0     IEEE 802.11  ESSID:off/any  Mode: Managed  Access Point: Not-Associated   Tx-Power=19 dBm Retry short limit:7   RTS thr:off   Fragment thr:off Power Management:off`
   - Confirmed network management uses `iwd` and `systemd-networkd` (no `NetworkManager` or `wpa_supplicant`).
   - Ran `inxi -N` to confirm the driver as `bcma-pci-bridge` a bus driver.
   - Reviewed `journalctl -u iwd.service` logs, showing repeated connection attempts and deauthentications (reasons 1 JN4).
   - Disabling IPv6 (`ipv6.disable=1`) had no effect.
   - Disabling power-saving modes had no effect.
   - Testing with different kernels and Linux distributions yielded the same issue.
   - Only this device is affected, regardless of distance from the router.
  
2. **Driver Troubleshooting Attempt** (2025-08-02):
   - To be included...
