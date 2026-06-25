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

**Step 2 — Find the matching nexmon patch**

Browse `patches/` in the [nexmon repo](https://github.com/seemoo-lab/nexmon/tree/master/patches) and look for a directory named after your chip, then your firmware version:

```
patches/
  bcm43430a1/        ← CardputerZero / RPi 3B / Pi Zero W
    7_45_41_26/      ← firmware version (dots → underscores)
  bcm43455c0/        ← RPi 3B+ / RPi 4
    7_45_154/
    7_45_189/
    7_45_206/
    7_45_234_4ca95bb_CY/
    7_45_241/        ← latest available as of mid-2026
```

If your firmware version has no matching directory, nexmon cannot be used safely — see Option B.

> **Risk warning — read before proceeding.** nexmon replaces the kernel driver and the firmware binary for your onboard WiFi chip. If the install fails, or if you're on an unsupported firmware version, **`wlan0` will stop working** and you may need physical access to the device (keyboard/monitor or serial console) to restore it. Always make sure you can reach the Pi without WiFi before running any of the steps below.

nexmon officially supports kernels up to **6.6**. Kernel 6.18 is supported via the [flintdevices fork](#rpi-3b-kernel-618-path) below. Other kernel versions are unsupported.

**Step 3 — Install (CardputerZero and kernel ≤ 6.6)**

Replace `<chip>` and `<firmware>` with the values you found:

```bash
sudo apt update
sudo apt install raspberrypi-kernel-headers git libgmp3-dev gawk qpdf bison flex make autoconf libtool texinfo
git clone https://github.com/seemoo-lab/nexmon
cd nexmon && source setup_env.sh
cd patches/<chip>/<firmware>/nexmon
make && make backup-firmware && make install-firmware
reboot
```

**CardputerZero example** (BCM43430A1, firmware 7.45.41.26):
```bash
cd patches/bcm43430a1/7_45_41_26/nexmon
make && make backup-firmware && make install-firmware
```

> **If this fails**, the most common cause is a firmware version mismatch. Run `dmesg | grep "Firmware:"` after the failed reboot and compare against the patch directory name. If there is no matching patch, use Option B.

---

#### RPi 3B+ — kernel 6.18 path {#rpi-3b-kernel-618-path}

Recent Raspberry Pi OS ships **firmware `7.45.265`**, which has no nexmon patch in the upstream repo. On top of that, the upstream nexmon `brcmfmac` kernel module does not compile on kernel 6.18 (timer API and cfg80211 signature changes). Both problems are fixed in the [flintdevices fork](https://github.com/flintdevices/raspberry-pi-4-wifi-csi-pi-os-bookworm).

> **Risk reminder — this can fail and break WiFi.** This replaces `wlan0`'s firmware with a nexmon-patched binary (`7.45.189`) and installs a patched DKMS driver on top of the kernel. The DKMS build targets a specific kernel version; if your kernel is updated before you run the script, or if the build fails for any reason, `wlan0` may stop working. Have a wired connection (eth0) or physical access ready before you start.

```bash
# Prerequisites: eth0 active (wired), ~200 MB free on /
sudo apt update && sudo apt install -y git

git clone https://github.com/flintdevices/raspberry-pi-4-wifi-csi-pi-os-bookworm.git
cd raspberry-pi-4-wifi-csi-pi-os-bookworm

# Read install.sh before running — it modifies /lib/firmware and installs a DKMS driver
sudo bash install.sh
sudo reboot
```

What `install.sh` does:
- Installs Kali's `brcmfmac-nexmon-dkms` 6.12.2 DKMS package (auto-rebuild on kernel upgrades)
- Applies `driver/kernel-6.18-porting.patch` and triggers a DKMS rebuild (auto-detected for kernel ≥ 6.16)
- Replaces the onboard firmware with a nexmon-patched `7.45.189` binary
- Installs `nexutil` (control utility for nexmon features)

After reboot, verify monitor mode works:
```bash
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
iw dev wlan0 info   # should show "type monitor"
```

If `iw` reports `nl80211 not found` or the set-type command fails, the driver did not load correctly — check `dmesg | grep brcmfmac` and `dkms status brcmfmac-nexmon`.

---

**Limitation (all nexmon paths):** while nexmon is active, `wlan0` cannot connect to networks or act as an AP — single radio, can't do both simultaneously.

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
