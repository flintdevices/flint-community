<div align="center">
  <img src="assets/flint_logo.png" alt="Flint" width="320" />
  <h1>Flint | Offensive Security Firmware</h1>
</div>

Offensive security firmware for the M5Stack CardputerZero, written in Python.
Pure Python stack with a pygame emulator for development and a framebuffer/evdev runtime for hardware targets.

# flint-community

This is the public issue tracker and feedback hub for **Flint** — offensive security firmware for Linux hardware.

The Flint source is closed, but this space is open. Use it to report bugs, request features, ask questions, or share ideas.

---

## Hardware Prerequisites

Several WiFi tools (deauth, beacon monitor, PMKID/handshake capture, probe scan, etc.) require monitor mode. The native WiFi chip on most Raspberry Pi hardware does **not** support monitor mode out of the box. You need one of the following:

### Option A — nexmon (no extra hardware)

nexmon patches the onboard WiFi firmware to add monitor mode support. Whether it works on your device depends on three variables that must all match a known-good combination:

| Variable | Why it matters |
|---|---|
| **WiFi chip** | Determines which firmware binary nexmon patches |
| **Firmware version** | The patch is binary and offset-specific — wrong version = bricked firmware |
| **Kernel version** | nexmon recompiles the `brcmfmac` kernel module; its internal API changes between kernel versions |

**Step 1 — Discover your variables**

```bash
# WiFi chip and firmware version
dmesg | grep -E "brcmf_c_preinit|brcmf_fw_alloc"

# Kernel version
uname -r
```

Example output on **CardputerZero** (BCM43430A1):
```
brcmfmac: brcmf_fw_alloc_request: using brcm/brcmfmac43430-sdio for chip BCM43430/1
brcmfmac: brcmf_c_preinit_dcmds: Firmware: BCM43430/1 wl0: ... version 7.45.41.26 ...
6.6.28+rpt-rpi-v8
```

Example output on **RPi 3B+** (BCM43455):
```
brcmfmac: brcmf_fw_alloc_request: using brcm/brcmfmac43455-sdio for chip BCM4345/6
brcmfmac: brcmf_c_preinit_dcmds: Firmware: BCM4345/6 wl0: ... version 7.45.265 ...
6.18.33+rpt-rpi-v8
```

> **Risk warning — read before proceeding.** nexmon replaces the kernel driver and the firmware binary for your onboard WiFi chip. If the install fails, **`wlan0` will stop working** and you may need physical access (keyboard/monitor or serial console) to restore it. Always make sure you can reach the Pi without WiFi before running any of the steps below.

**Step 2 — Install**

Use the [flintdevices/nexmon-install](https://github.com/flintdevices/raspberry-pi-4-wifi-csi-pi-os-bookworm) installer. It handles firmware selection, kernel driver patching, and DKMS setup automatically.

```bash
# Prerequisites: wired connection (eth0) active, ~200 MB free on /
sudo apt update && sudo apt install -y git

git clone https://github.com/flintdevices/raspberry-pi-4-wifi-csi-pi-os-bookworm.git
cd raspberry-pi-4-wifi-csi-pi-os-bookworm

# Read install.sh before running — it modifies /lib/firmware and installs a DKMS driver
sudo bash install.sh
sudo reboot
```

What `install.sh` does:
- Installs a nexmon-patched firmware for your chip
- Installs a DKMS kernel driver (auto-rebuild on kernel upgrades)
- Auto-detects kernel ≥ 6.16 and applies the necessary API porting patch
- Installs `nexutil` (control utility for nexmon features)

After reboot, verify monitor mode works:
```bash
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
iw dev wlan0 info   # should show "type monitor"
```

If the set-type command fails, check `dmesg | grep brcmfmac` and `dkms status brcmfmac-nexmon` for errors.

**Validating monitor mode**

After setting monitor mode, confirm that raw 802.11 frames are being captured:

```bash
sudo apt install -y tcpdump          # if not already installed
sudo tcpdump -i wlan0 -c 5 -e 2>&1
```

Expected output: 5 lines with synthetic MAC addresses and `[|llc]` — these are 802.11 frames (beacons, probes, data) captured from the air and wrapped by the nexmon driver. Example:

```
20:29:15.915212  00:00:06:10:02:00 > 00:00:00:00:04:00, 802.3, length 528:  [|llc]
20:29:15.922022  00:00:06:10:16:00 > 00:00:00:00:04:00, 802.3, length 46:   [|llc]
```

If you see 0 packets, the driver did not load correctly — check `dmesg | grep brcmfmac`.

> **CardputerZero on kernel 6.18 — coming soon.** The same kernel driver changes needed for RPi 3B+ will almost certainly be required for BCM43430A1 as well. We haven't validated this yet (no hardware running 6.18 available). Until then, CardputerZero requires kernel ≤ 6.6. Use Option B if your kernel is newer.

**Limitation:** while nexmon is active, `wlan0` cannot connect to networks or act as an AP — single radio, can't do both simultaneously.

After reboot, Flint auto-detects nexmon and offers enable/disable from **WiFi › Settings › Monitor Setup**.

### Option B — USB WiFi dongle (recommended for RPi 3B+ and power users)

No firmware changes required. Native WiFi remains fully usable in parallel. This is the safest option regardless of device.

- **Supported adapters:** MT7612U (recommended), AR9271
- **Drivers:** `mt76x2u`, `ath9k_htc` — included in Raspberry Pi OS
- Plug in → Flint detects automatically. No configuration needed.

---

Flint's built-in UI surfaces this guide too: **WiFi › Settings › Monitor Setup**.

---

## Opening an issue

Just open one. No template required.

If you want to give more context, here are some things that help:

- **Bug report** — what you did, what you expected, what actually happened. Device and Flint version if you know it.
- **Feature request** — what you'd like to do, and why the current tools don't cover it.
- **Question** — anything about usage, hardware compatibility, or how something works.

If you're not sure which category fits, don't worry about it. Open the issue and we'll figure it out together.

→ [Open an issue](https://github.com/flintdevices/flint-community/issues/new)

---

## Links

- [flintdevices.dev](https://flintdevices.dev) — landing page, install instructions, screenshots
- [flintdevices](https://github.com/flintdevices) — GitHub organization

---

## Scope

This repo is for feedback only — no source code lives here. Pull requests won't be merged.

Issues may be closed without comment if they're out of scope (e.g. requests for help cracking networks you don't own).
