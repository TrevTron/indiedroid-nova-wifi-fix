# Indiedroid Nova WiFi Fix

Fixes the **RTL8821CS (BL-M8821) WiFi** on the Indiedroid Nova running Debian 12 or Ubuntu 24.04 rockchip images.

This resolves the long-standing issue where WiFi shows as **"unavailable"** in NetworkManager — reported in [Joshua-Riek/ubuntu-rockchip#1007](https://github.com/Joshua-Riek/ubuntu-rockchip/issues/1007) (open since Aug 2024, closed as "not planned" Feb 2025).

## The Problem

After booting the Indiedroid Nova, `nmcli dev status` shows:

```
DEVICE     TYPE      STATE         CONNECTION
wlan0      wifi      unavailable   --
```

The WiFi interface exists but NetworkManager refuses to manage it. `dmesg` shows SDIO timeout errors:

```
rtw_8821cs mmc2:0001:1: sdio read32 failed (0x11080): -110
rtw_8821cs mmc2:0001:1: failed to load firmware
rtw_8821cs: probe of mmc2:0001:1 failed with error -22
```

## Root Cause

**Two problems stacked on top of each other:**

1. **SDIO timing bug** — The `rtw88_8821cs` driver loads too early during boot before the SDIO bus is fully ready. The previous workaround (unload/reload driver modules) was known but incomplete.

2. **Missing `wpasupplicant` package** — This is the part nobody caught. NetworkManager **cannot manage WiFi interfaces** without `wpa_supplicant` installed. On minimal Debian/Ubuntu server images for this board, it's not included by default.

## Quick Fix

```bash
git clone https://github.com/TrevTron/indiedroid-nova-wifi-fix.git
cd indiedroid-nova-wifi-fix
chmod +x fix-wifi.sh
./fix-wifi.sh
sudo reboot
```

After reboot:

```bash
nmcli dev wifi list
sudo nmcli dev wifi connect "YourSSID" password "YourPassword"
```

## Manual Steps

### Step 1: Install required packages

```bash
sudo apt update
sudo apt install -y firmware-realtek wpasupplicant wireless-regdb iw rfkill
```

| Package | Why |
|---------|-----|
| `firmware-realtek` | Provides `rtw8821c_fw.bin` (WiFi) and `rtl8168h-2.fw` (Ethernet) |
| `wpasupplicant` | **Critical** — NetworkManager needs this to manage WiFi |
| `wireless-regdb` | Regulatory database for WiFi channels |
| `iw` / `rfkill` | Diagnostic tools |

### Step 2: Create systemd service

```bash
sudo tee /etc/systemd/system/fix-wifi.service > /dev/null << 'EOF'
[Unit]
Description=Reload RTL8821CS WiFi driver (workaround for SDIO timing bug)
After=network-pre.target
Before=NetworkManager.service
Wants=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'rmmod rtw88_8821cs rtw88_8821c rtw88_sdio rtw88_core 2>/dev/null; sleep 2; modprobe rtw88_8821cs'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable fix-wifi.service
```

### Step 3: Reboot and connect

```bash
sudo reboot
# After reboot:
nmcli dev wifi list
sudo nmcli dev wifi connect "YourSSID" password "YourPassword"
```

## Chip Info

| Field | Value |
|-------|-------|
| Module | BL-M8821 |
| Actual chip | Realtek RTL8821CS |
| SDIO Vendor ID | `024C` (Realtek) |
| SDIO Device ID | `C821` |
| Interface | SDIO via MMC2 |
| Driver | `rtw88_8821cs` |
| Firmware | `rtw8821c_fw.bin` (v24.11.0) |
| Capabilities | Dual-band WiFi (2.4/5GHz) 802.11ac + Bluetooth 5.0 |

## Tested On

- Indiedroid Nova (RK3588S, 16GB RAM)
- Debian 12 Bookworm, Kernel 6.1.0-1023-rockchip
- Should also work on Ubuntu 24.04 Noble (same kernel/driver stack)

## References

- [Joshua-Riek/ubuntu-rockchip#1007](https://github.com/Joshua-Riek/ubuntu-rockchip/issues/1007) — Original issue report
- [Ameridroid Indiedroid Nova](https://ameridroid.com/products/indiedroid-nova?ref=ioqothsk) — Where to buy

## License

MIT
